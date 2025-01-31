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

![vswitch_trunk](https://github.com/user-attachments/assets/e965aa58-a2a2-4959-8e92-db576cd6cdad)
  


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
  
![iso_add](https://github.com/user-attachments/assets/693f2b7d-e804-482c-bb63-ba1de835f2fc)

