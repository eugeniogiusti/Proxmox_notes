# Guide: Configuring a Trunk Interface on Proxmox 

This guide covers two important aspects:
1. **Configuring a network interface as a trunk on a Proxmox server.**
2. **Mounting an LVM disk with ISO images for use in Proxmox.**

---

## **1. Configuring a Trunk Network Interface on Proxmox**

A trunk interface allows Proxmox to handle multiple VLANs on a single physical interface. Let's assume the network card is `enp3s0`, and you want to use it for VLANs.

### **Steps:**

### **1.1 Configure a VLAN-Aware Linux Bridge**
1. Go to **Proxmox Web UI** → **Datacenter** → **Proxmox Node** → **Network**.
2. Click **Create** → **Linux Bridge** and configure:
   - **Name:** `vmbr1`
   - **Bridge ports:** `enp3s0`
   - **VLAN aware:** ✅ (Enabled)
   - **IP Address / Gateway:** Leave empty if this interface is only for VMs.
   - **Save and apply changes.**

### **1.2 Configure the VM to Use a VLAN**
When creating or modifying a VM:
- Go to **Hardware** → **Network Device**.
- Select `vmbr1` as the bridge.
- In the **VLAN Tag** field, enter the VLAN ID where the VM should operate.
- Save the changes and start the VM.

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
mkdir -p /mnt/iso_storage
mount /dev/<volume-group-name>/<logical-volume-name> /mnt/iso_storage
```

### **2.3 Configure Automatic Mounting**
Add the following line to `/etc/fstab` for automatic mounting at startup:
```bash
/dev/<volume-group-name>/<logical-volume-name> /mnt/iso_storage ext4 defaults 0 2
```

### **2.4 Add ISO Storage to Proxmox**
1. Go to **Proxmox Web UI** → **Datacenter** → **Storage**.
2. Click **Add** → **Directory**.
3. Set:
   - **ID:** `iso_storage`
   - **Directory Path:** `/mnt/iso_storage`
   - **Content:** `ISO Image`
---

Now your Proxmox server is configured with a VLAN-aware trunk interface, and the disk with ISOs is mounted and accessible. 🚀

