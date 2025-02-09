# Guide: Configuring a Trunk Interface on Proxmox 

--

## **1. Configuring a Trunk Network Interface on Proxmox**

A trunk interface allows Proxmox to handle multiple VLANs on a single physical interface. Let's assume the network card is `enp3s0`, and you want to use it for VLANs.

### **Steps:**

### **1.1 Configure a VLAN-Aware Linux Bridge**
1. Go to **Proxmox Web UI** → **Datacenter** → **Proxmox Node** → **Network**.
2. Click **Create** → **Linux Bridge** and configure:
   - **Name:** `vmbr1`
   - **Bridge ports:** `eth0`
   - **VLAN aware:** ✅ (Enabled)
   - **IP Address / Gateway:** Leave empty if this interface is only for VMs.
   - **Save and apply changes.**

![ksnip_20250208-181013](https://github.com/user-attachments/assets/28c25066-2e7d-429a-9435-e67119fcbf5a)

  


### **1.2 Configure the VM to Use a VLAN**
When creating or modifying a VM:
- Go to **Hardware** → **Network Device**.
- Select `vmbr1` as the bridge.
- In the **VLAN Tag** field, enter the VLAN ID where the VM should operate.
- Save the changes and start the VM.

![ksnip_20250131-221000](https://github.com/user-attachments/assets/a5d62ee3-abe2-4dfb-be34-924e115ba07c)


---

# Mounting a Disk with ISO in Proxmox

## **2. Mounting an LVM Disk with ISO in Proxmox**

If you have a disk with an LVM volume that contains ISO images and want to mount it on Proxmox, follow these steps:

### **2.1 Identify the LVM Volume**
Run:
```bash
lsblk
```
To see the LVM volume name, execute:
```bash
vgscan
lvscan
```
If the volume is inactive, activate it with:
```bash
vgchange -ay <volume-group-name>
```

### **2.2 Create the Mount Point and Mount the Volume**
```bash
mkdir -p /iso
mount /dev/<volume-group-name>/<logical-volume-name> /iso
```

### **2.3 Configure Automatic Mounting**
Add the following line to `/etc/fstab` for automatic mounting at startup:
```bash
/dev/<volume-group-name>/<logical-volume-name> /iso ext4 defaults 0 2
```

### **2.4 Add ISO Storage to Proxmox**
1. Go to **Proxmox Web UI** → **Datacenter** → **Storage**.
2. Click **Add** → **Directory**.
3. Set:
   - **ID:** `iso_images`
   - **Directory Path:** `/iso`
   - **Content:** `ISO Image`
  
![iso_add](https://github.com/user-attachments/assets/693f2b7d-e804-482c-bb63-ba1de835f2fc)



# Setting Up ZFS Storage with RAID-Z on Proxmox

This guide explains how to configure **ZFS storage with RAID-Z (RAID5 equivalent)** on a **Proxmox server** using **three disks**.

---

## **Prerequisites**
- A running **Proxmox VE** server.
- Three **unused disks** (e.g., `/dev/sdb`, `/dev/sdc`, `/dev/sdd`).
- At least **8GB of RAM** (ZFS recommends 1GB per TB of storage).

---

## **Step 1: Identify Available Disks**
Run the following command to list available disks:

```sh
lsblk
```

Example output:
```
sda      500G  (OS Disk)
sdb      1TB   (New Disk 1)
sdc      1TB   (New Disk 2)
sdd      1TB   (New Disk 3)
```
Ensure the disks are **not** mounted and do not contain important data.

---

## **Step 2: Create the ZFS Pool**
To create a ZFS pool named `zpool_vmstorage` with RAID-Z (RAID5 equivalent):

```sh
zpool create -f zpool_vmstorage raidz /dev/sdb /dev/sdc /dev/sdd
```

This will:
- Create a **RAID-Z (RAID5) pool** with redundancy (1 disk failure tolerance).
- Name the pool `zpool_vmstorage`.
- Use `/dev/sdb`, `/dev/sdc`, and `/dev/sdd`.

To verify the pool:
```sh
zpool status
```

---

## **Step 3: Configure ZFS in Proxmox**
### **Add the ZFS Storage**
1. Open the **Proxmox Web UI**.
2. Go to **Datacenter → Storage**.
3. Click **Add → ZFS**.
4. Set the following:
   - **ID:** `zfs_vmstorage`
   - **Pool:** `zpool_vmstorage`
   - **Thin provisioning:** ✅ Enabled
5. Click **Add**.

---

## **Step 4: Enable Compression (Optional, Recommended)**
ZFS supports compression, which saves space and improves performance. Enable LZ4 compression:
```sh
zfs set compression=lz4 zpool_vmstorage
```
Verify:
```sh
zfs get compression zpool_vmstorage
```

---

## **Step 5: Enable Deduplication (Optional, Use with Caution)**
If you have a **lot of duplicate data** and enough RAM, enable deduplication:
```sh
zfs set dedup=on zpool_vmstorage
```
Check status:
```sh
zfs get dedup zpool_vmstorage
```
❗ **Warning:** Deduplication requires a lot of RAM and should be enabled only if needed.

---

## **Step 6: Test Snapshots**
Snapshots allow quick recovery. To create a snapshot:
```sh
zfs snapshot zpool_vmstorage@snapshot1
```
List snapshots:
```sh
zfs list -t snapshot
```
Rollback to a snapshot:
```sh
zfs rollback zpool_vmstorage@snapshot1
```

---

## **Step 7: Enable TRIM (For SSDs)**
If you are using SSDs, enable TRIM for better performance:
```sh
zpool set autotrim=on zpool_vmstorage
```
Verify:
```sh
zpool get autotrim zpool_vmstorage
```

---

## **Step 8: Set Up Automatic Scrubbing**
Scrubbing checks for data integrity. Schedule a cron job:
```sh
crontab -e
```
Add this line to run scrubbing every Sunday at midnight:
```
0 0 * * 0 /sbin/zpool scrub zpool_vmstorage
```

---

## **Conclusion**
Your Proxmox server is now configured with **ZFS RAID-Z storage**, supporting:
✅ Snapshots  
✅ Compression  
✅ Redundancy (RAID5-like)  
✅ TRIM (for SSDs)  

You can now store **VMs and containers** safely on your ZFS pool.

---
### **Troubleshooting**
- Check the status of your pool:
  ```sh
  zpool status
  ```
- If a disk fails, replace it and run:
  ```sh
  zpool replace zpool_vmstorage <old-disk> <new-disk>
  ```
- If running out of space, add more disks:
  ```sh
  zpool add zpool_vmstorage /dev/sdX
  ```


# QEMU Guest Agent Installation Guide for Proxmox VE

This guide provides step-by-step instructions for installing and configuring the QEMU Guest Agent on both Linux and Windows virtual machines running on Proxmox VE.

## What is QEMU Guest Agent?

The QEMU Guest Agent is a daemon that runs inside the virtual machine and enables better integration between the host and guest, allowing for:
- Graceful shutdown and reboot
- File system quiescing for better snapshots
- System information reporting
- Network configuration
- Memory statistics reporting

## Prerequisites

- Running Proxmox VE installation
- Virtual machine with either Linux or Windows operating system
- Administrative access to the virtual machine
- Internet connection for package installation

## Enabling QEMU Guest Agent in Proxmox VE

1. Shut down the virtual machine
2. In the Proxmox web interface, select your VM
3. Go to "Hardware"
4. Click "Add" and select "QEMU Guest Agent"
5. Start the virtual machine

## Linux Installation

### Debian/Ubuntu-based Systems

```bash
# Update package list
sudo apt update

# Install QEMU Guest Agent
sudo apt install qemu-guest-agent

# Start the service
sudo systemctl start qemu-guest-agent

# Enable the service to start on boot
sudo systemctl enable qemu-guest-agent

# Verify the service status
sudo systemctl status qemu-guest-agent
```

### RHEL/CentOS/Fedora-based Systems

```bash
# Install QEMU Guest Agent
sudo dnf install qemu-guest-agent

# Start the service
sudo systemctl start qemu-guest-agent

# Enable the service to start on boot
sudo systemctl enable qemu-guest-agent

# Verify the service status
sudo systemctl status qemu-guest-agent
```

### Arch Linux

```bash
# Install QEMU Guest Agent
sudo pacman -S qemu-guest-agent

# Start the service
sudo systemctl start qemu-guest-agent

# Enable the service to start on boot
sudo systemctl enable qemu-guest-agent

# Verify the service status
sudo systemctl status qemu-guest-agent
```

## Windows Installation

### Method 1: VirtIO ISO

1. Download the latest VirtIO ISO from: https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio-win-gt-x64/
2. Add the ISO to your VM's CD/DVD drive in Proxmox
3. Mount the CD drive in Windows
4. Navigate to guest-agent/qemu-ga-x86_64.msi
5. Double-click the MSI file and follow the installation wizard
6. Restart the virtual machine

### Method 2: Chocolatey Package Manager

```powershell
# Install Chocolatey if not already installed
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

# Install QEMU Guest Agent
choco install qemu-guest-agent -y
```

## Verification

### Linux
To verify the guest agent is working:
```bash
sudo systemctl status qemu-guest-agent
```

Expected output should show "active (running)".

### Windows
1. Open Services (services.msc)
2. Look for "QEMU Guest Agent"
3. Status should show "Running"

## Troubleshooting

### Linux
1. Check if the service is running:
```bash
ps aux | grep qemu-ga
```

2. Check system logs:
```bash
journalctl -u qemu-guest-agent
```

### Windows
1. Check Windows Event Viewer for any errors
2. Verify the service is running:
```powershell
Get-Service QEMU-GA
```

### Useful link
Guide: https://pve.proxmox.com/wiki/Qemu-guest-agent
Video Linux:https://www.youtube.com/watch?v=ZpEYkA4C4jY
Video Windows:https://www.youtube.com/watch?v=0iTMk56V7hk
