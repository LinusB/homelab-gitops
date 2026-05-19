# Proxmox VE: Converged Hosting & System Architecture

This document outlines the technical feasibility, architectural design, and operational implementation of our hyperconverged Proxmox VE infrastructure. It specifically addresses the consolidation of persistent storage services (Nextcloud, Immich) alongside transient, resource-intensive workloads (SAP ASE, Kali Linux) on a single host with 32 GB of RAM.

---

## 1. Executive Summary & Architectural Context

The consolidation of services on a single hypervisor presents a classic optimization problem within hyperconverged infrastructures. Running persistent storage services (the "Service VM") alongside transient, high-demand workloads (e.g., SAP ASE, Kali Linux) requires precise resource orchestration.

Based on an analysis of the system architecture and ZFS memory management, tuning the Adaptive Replacement Cache (ARC) for a 32 GB RAM system is not just recommended, but **critical for system stability**. The assumption that transient workloads do not require static resource reservations is a fallacy in the context of ZFS and database-heavy applications. Without limiting the ARC, the host risks triggering an "Out-of-Memory" (OOM) state during the startup spikes of the SAP instance.

Furthermore, operating Kali Linux within the same logical network as private data violates the principle of "Least Privilege". This document details a **Zero-Trust architecture** utilizing Proxmox's kernel-level firewall to enforce strict microsegmentation without requiring additional hardware.

---

## 2. Memory Architecture & ZFS ARC Management

The primary challenge of using ZFS as the foundational filesystem for virtualization is the contention for Random Access Memory (RAM). To understand the necessity of ARC tuning, we must examine the interaction between the Linux kernel memory manager and the ZFS ARC.

### 2.1 ZFS ARC Dynamics under Linux

By design, ZFS on Linux (ZoL) allocates up to 50% of the available host memory for the Adaptive Replacement Cache (ARC). On a 32 GB system, the ARC can consume up to 16 GB. Theoretically, ZFS yields this memory back to the system when the kernel signals memory pressure.

However, in a virtualized environment under Proxmox, this reclamation process is not instantaneous. When a VM is booted, the QEMU/KVM process requests its allocated memory almost immediately. If the RAM is saturated with "Dirty Pages" or metadata in the ARC, ZFS must flush (evict) these to the disk before the memory becomes available to the KVM process.

!!! warning "The Race Condition: Transient SAP Workloads"
    SAP ASE and its Java runtime allocate massive, contiguous memory blocks upon startup. 
    If the ARC is fully utilizing its 16 GB allocation, and a transient VM suddenly requests 12 GB, a race condition occurs. If ZFS cannot shrink the ARC fast enough, the **Linux OOM (Out-of-Memory) Killer** will intervene to save the host, often terminating the processes consuming the most memory—which is typically the database of the Service VM or the SAP instance itself.

### 2.2 Resource Budgeting & Tuning Rationale

To guarantee stability, memory must be statically budgeted. The following table illustrates the component requirements used to calculate the strict ARC limit.

| Infrastructure Component | Role | Estimated Memory Requirement | Allocation Rationale |
| :--- | :--- | :--- | :--- |
| **Proxmox VE (Host)** | Hypervisor | 2 GB | Baseline for Linux Kernel, Cluster API, and Web GUI. |
| **VM 100: Service Monolith** | Persistent | 8 - 10 GB | Nextcloud (PHP/Redis/MariaDB) + Immich (Machine Learning/Postgres). |
| **Transient VMs (e.g., SAP)** | Ephemeral | 8 - 12 GB | SAP ASE minimum requirements for development environments. |
| **VM 999: Kali Linux** | Ephemeral | 4 GB | Standard allocation for penetration testing toolkits. |
| **System Buffer** | Reserve | 2 GB | Buffer to prevent swapping during temporary load spikes. |
| **ZFS ARC Limit (Cache)** | System Cache | **4 - 6 GB** | **Resulting strict limit to prevent host overcommitment.** |

!!! note "Architectural Rationale: Why 4-6 GB?"
    Adding the requirements of the Service VM (10 GB), SAP VM (12 GB), and Host (2 GB) consumes 24 GB. If the ARC defaults to 16 GB, the system requires 40 GB, leading to massive overcommitment. However, setting the ARC too low (< 2 GB) severely degrades performance, especially for metadata-heavy operations in Nextcloud. A hard limit of 6 GB represents the optimal compromise.

### 2.3 Implementation: ARC Tuning & Host Preparation

The following scripts are consolidated for your MkDocs structure. They should be executed on the Proxmox host to prepare the repositories and enforce the ARC limits.

```bash title="scripts/proxmox/01_host_preparation.sh"
#!/bin/bash
# 1. Disable Enterprise Repo & Enable No-Subscription
sed -i 's/^deb/#deb/g' /etc/apt/sources.list.d/pve-enterprise.list
echo "deb [http://download.proxmox.com/debian/pve](http://download.proxmox.com/debian/pve) bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
echo "deb [http://download.proxmox.com/debian/ceph-quincy](http://download.proxmox.com/debian/ceph-quincy) bookworm no-subscription" > /etc/apt/sources.list.d/ceph.list

# 2. Update and install critical headers & firewall
apt update && apt dist-upgrade -y
apt install pve-headers proxmox-firewall -y
```

```bash title="scripts/proxmox/02_zfs_arc_tuning.sh"
#!/bin/bash
# ZFS ARC Tuning for 32GB Host
# Min: 2GB (2147483648 Bytes), Max: 6GB (6442450944 Bytes)

cat <<EOF > /etc/modprobe.d/zfs.conf
options zfs zfs_arc_min=2147483648
options zfs zfs_arc_max=6442450944
EOF

# Update initramfs to apply on next boot
update-initramfs -u
# NOTE: A system reboot is required for these changes to take effect.
```

---

## 3. Storage Design: Multi-Disk Architecture

Combining Nextcloud and Immich into a single VM dramatically simplifies disaster recovery (enabling atomic snapshots of the entire state). However, it creates a highly complex I/O profile with competing write patterns:

1. **Databases (OLTP):** MariaDB (Nextcloud) and PostgreSQL (Immich) generate small, random write blocks (typically 8 KB or 16 KB).
2. **Blob Storage:** Photos, videos, and documents represent large, sequential data streams.

### 3.1 ZFS Write Amplification

ZFS organizes data into "Volblocks" (for Zvols). If a virtual disk uses a standard `volblocksize` of 64 KB, and a database writes an 8 KB page, ZFS must read the full 64 KB block, modify the 8 KB segment, and rewrite the entire 64 KB block. 

This **Read-Modify-Write (RMW)** cycle doubles the I/O load and drastically accelerates SSD wear. Conversely, using an 8 KB `volblocksize` for massive photo libraries generates extreme metadata overhead, choking the ARC and reducing throughput.

### 3.2 Optimized Storage Layout

To resolve this, we implement a Multi-Disk strategy, assigning specific ZFS block sizes to dedicated virtual disks based on their workload.

| Virtual SCSI Disk | Guest Mountpoint | ZFS Volblocksize | Guest Filesystem | Architectural Rationale |
| :--- | :--- | :--- | :--- | :--- |
| `scsi0 (32GB)` | `/` (Root) | 16 KB (Default) | `ext4` | Standard OS workload; primarily sequential reading of binaries. |
| `scsi1 (64GB)` | `/var/lib/docker` | **16 KB** | `ext4` | Highly optimized for PostgreSQL/InnoDB 16 KB pages. Completely eliminates RMW cycles. |
| `scsi2 (1TB)` | `/mnt/data` | **64 KB - 1 MB** | `xfs` / `ext4` | Optimized for large binary blobs (Photos/Files). Massively reduces metadata overhead. |

```bash title="scripts/proxmox/03_vm100_provisioning.sh"
#!/bin/bash
# VM 100: Service Monolith Provisioning

# Create base VM with AVX passthrough (cpu: host) for Machine Learning
qm create 100 --name "Service-Monolith" \
  --memory 8192 --balloon 2048 \
  --cores 4 --sockets 1 --cpu host \
  --net0 virtio,bridge=vmbr0,firewall=1 \
  --ostype l26 --agent enabled=1 --scsihw virtio-scsi-single

# Disk 0: OS (Default Blocksize)
pvesm alloc local-zfs 100 vm-100-disk-0 32G
qm set 100 --scsi0 local-zfs:vm-100-disk-0,discard=on,ssd=1,iothread=1

# Disk 1: Database & Docker Volumes (Strict 16k Blocksize)
zfs create -V 64G -o volblocksize=16k rpool/data/vm-100-disk-1
qm set 100 --scsi1 local-zfs:vm-100-disk-1,discard=on,ssd=1,iothread=1

# Disk 2: Blob Storage (Large Blocksize)
zfs create -V 1T -o volblocksize=64k rpool/data/vm-100-disk-2
qm set 100 --scsi2 local-zfs:vm-100-disk-2,discard=on,ssd=1,iothread=1

qm set 100 --boot c --bootdisk scsi0
```

!!! info "SSD Longevity via TRIM"
    The parameters `discard=on` and `ssd=1` are essential. They signal to the guest OS that the underlying storage is flash-based, enabling TRIM operations. This is vital for the longevity of the physical NVMe drives on the Proxmox host.

---

## 4. Security Architecture: Zero-Trust Isolation

Kali Linux is an offensive security distribution containing tools designed for network reconnaissance, Man-in-the-Middle (MitM) attacks, and exploitation. Operating Kali within the same flat network (Layer 2 broadcast domain) as private homelab data presents severe security vectors.

### 4.1 Threat Vectors in Flat Networks

* **Accidental Exposure:** An inadvertently misconfigured vulnerability scan (e.g., OpenVAS/Nmap) targeting the local subnet could destabilize Nextcloud or trigger `fail2ban` lockouts.
* **Lateral Movement:** If the Kali VM is compromised (e.g., via malicious scripts or exposed listeners), it becomes a Bastion Host. Attackers can "Live off the Land" utilizing pre-installed offensive tools to attack the Service VM.
* **ARP Spoofing:** Operating on the same bridge (`vmbr0`) allows Kali to intercept traffic between the Service VM and the gateway, compromising unencrypted internal communications.

### 4.2 Proxmox Firewall Microsegmentation

To mitigate this, Kali Linux is treated as an **Untrusted Entity**. We utilize the kernel-level Proxmox Firewall (nftables/iptables) to enforce a Zero-Trust perimeter.

!!! abstract "Visualizing Zero-Trust Network Segmentation"
    `![Network Segmentation Architecture](../assets/images/excalidraw_kali_isolation_placeholder.png)`

The firewall enforces the following logic:
1. **Service VM:** Placed in the "Trusted LAN". Accepts traffic from the local network but drops Kali's IP explicitly.
2. **Kali VM:** Placed in an isolated logical zone. It is permitted egress to the global internet (`0.0.0.0/0`), but **all traffic directed at RFC1918 private subnets is dropped at the hypervisor level.**

```ini title="configs/proxmox/firewall/cluster.fw"
[group service_protection]
# Allow Web (80/443) from LAN
IN ACCEPT -p tcp -dport 80 -log nolog
IN ACCEPT -p tcp -dport 443 -log nolog

# Allow SSH ONLY from Admin IPs
IN ACCEPT -p tcp -dport 22 -source 192.168.1.10-192.168.1.20 -log nolog

# EXPLICITLY DROP Kali VM
IN DROP -source 192.168.1.99 -log warning
```

```ini title="configs/proxmox/firewall/101.fw (Kali VM)"
enable: 1
policy_in: ACCEPT
policy_out: ACCEPT

# Egress Filtering: Block access to all local subnets (RFC1918)
OUT DROP -dest 192.168.0.0/16 -log warning
OUT DROP -dest 10.0.0.0/8 -log warning
OUT DROP -dest 172.16.0.0/12 -log warning

# Allow all other traffic (Internet)
OUT ACCEPT -dest 0.0.0.0/0
```

---

## 5. Operations & Disaster Recovery

Consolidating services into VM 100 drastically simplifies our backup topology. Rather than synchronizing database dumps and file systems manually, we orchestrate backups directly via the hypervisor.

### 5.1 Proxmox Backup Server (PBS) vs. Vzdump

For a VM managing over 1 TB of data (Immich photo libraries), the integrated `vzdump` utility is highly inefficient, as it requires reading and compressing a Full Backup image during every run. 

**PBS is the mandatory standard for this architecture due to:**
* **Client-side Deduplication:** PBS processes data in chunks. If 1 GB of photos are added to a 1 TB VM, only the modified 1 GB delta is transmitted over the network.
* **Dirty Bitmaps:** Proxmox tracks modified storage blocks in RAM via QEMU "Dirty Bitmaps". During an incremental backup, PBS instantly identifies which sectors to read, reducing backup times from hours to mere seconds.

### 5.2 Consistent Snapshots

Because Nextcloud and Immich operate 24/7, the VM cannot be shut down for backups.

```bash title="scripts/proxmox/04_backup_trigger.sh"
# Manual PBS backup trigger forcing snapshot mode
vzdump 100 --mode snapshot --compress zstd --storage pbs-storage
```

!!! note "Rationale: Atomic Consistency"
    Using `--mode snapshot` leverages the QEMU Guest Agent to briefly freeze the guest filesystem (`fs-freeze`). This ensures that the PostgreSQL database on Disk 1 and the physical photo files on Disk 2 are captured in the exact same millisecond, preventing relational corruption upon restoration.