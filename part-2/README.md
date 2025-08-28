# Create and Configure VirtualBox VMs with VBoxManage (BIOS/Legacy)

This guide provides instructions to create multiple VMs (tools, k8s-master, k8s-worker, kali-linux) and boot them from a custom unattended Debian ISO.

**Note:** We force BIOS firmware to use the ISOLINUX flow validated for unattended installs.

## 1. Prerequisites

- **VirtualBox 7.x** (VBoxManage in PATH)
- **Custom ISO** from the Preseed ISO guide, e.g.:
  ```
  /vm/iso/debian-12.11.0-amd64-netinst-custom.iso
  ```

## 2. ⚠️ (Optional) Remove All VMs — DANGER ZONE

**WARNING:** This script will permanently delete ALL VMs and their data. Use with extreme caution.

```bash
#!/bin/bash

# Power off running VMs and unregister all VMs (irreversible)
for vm in $(VBoxManage list vms | awk -F\" '{print $2}'); do
  if VBoxManage showvminfo "$vm" | grep -q "running (since"; then
    VBoxManage controlvm "$vm" poweroff
  fi
  VBoxManage unregistervm "$vm" --delete
done
```

## 3. Create VM: tools

Set the ISO path variable:

```bash
OUT_ISO="/vm/iso/debian-13.0.0-amd64-netinst-custom.iso"
```

Create and configure the tools VM:

```bash
VBoxManage createvm --name "tools" --ostype Debian_64 --basefolder /vm --register
VBoxManage modifyvm "tools" --firmware bios --boot1 dvd --boot2 disk --boot3 none --boot4 none
VBoxManage modifyvm "tools" --memory 6144 --cpus 3 --audio-driver none --graphicscontroller vmsvga --vram 16
VBoxManage createhd --filename /vm/tools/tools.vdi --size 35000 --variant Standard
VBoxManage storagectl "tools" --name "SATA Controller" --add sata --controller IntelAhci
VBoxManage storageattach "tools" --storagectl "SATA Controller" --port 0 --device 0 --type hdd --medium /vm/tools/tools.vdi --nonrotational on --hotpluggable on
VBoxManage modifyvm "tools" --macaddress1 0800272762D6
VBoxManage setextradata "tools" "VBoxInternal/Devices/ahci/0/Config/Port0/IsSSD" 1
VBoxManage modifyvm "tools" --nic1 bridged --bridgeadapter1 enp5s0
VBoxManage storagectl "tools" --name "IDE Controller" --add ide
VBoxManage storageattach "tools" --storagectl "IDE Controller" --port 0 --device 0 --type dvddrive --medium "$OUT_ISO"
```

Start the VM (headless or GUI):

```bash
# Headless mode (no GUI)
VBoxManage startvm "tools" --type headless

# GUI mode (remove --type headless)
VBoxManage startvm "tools"
```

### VM Specifications: tools
- **Memory:** 6144 MB (6 GB)
- **CPUs:** 3
- **Storage:** 35 GB VDI
- **Network:** Bridged adapter (enp5s0)
- **MAC Address:** 08:00:27:27:62:D6

## 4. Create VM: k8s-master

Create and configure the Kubernetes master VM:

```bash
VBoxManage createvm --name "k8s-master" --ostype Debian_64 --basefolder /vm --register
VBoxManage modifyvm "k8s-master" --firmware bios --boot1 dvd --boot2 disk --boot3 none --boot4 none
VBoxManage modifyvm "k8s-master" --memory 10240 --cpus 5 --audio-driver none --graphicscontroller vmsvga --vram 16
VBoxManage createhd --filename /vm/k8s-master/k8s-master.vdi --size 40000 --variant Standard
VBoxManage storagectl "k8s-master" --name "SATA Controller" --add sata --controller IntelAhci
VBoxManage storageattach "k8s-master" --storagectl "SATA Controller" --port 0 --device 0 --type hdd --medium /vm/k8s-master/k8s-master.vdi --nonrotational on --hotpluggable on
VBoxManage modifyvm "k8s-master" --macaddress1 0800279E473A
VBoxManage modifyvm "k8s-master" --nic1 bridged --bridgeadapter1 enp5s0
VBoxManage storagectl "k8s-master" --name "IDE Controller" --add ide
VBoxManage storageattach "k8s-master" --storagectl "IDE Controller" --port 0 --device 0 --type dvddrive --medium "$OUT_ISO"
```

Start the VM:

```bash
VBoxManage startvm "k8s-master" --type headless
```

### VM Specifications: k8s-master
- **Memory:** 10240 MB (10 GB)
- **CPUs:** 5
- **Storage:** 40 GB VDI
- **Network:** Bridged adapter (enp5s0)
- **MAC Address:** 08:00:27:9E:47:3A

## 5. Create VM: k8s-worker

Create and configure the Kubernetes worker VM:

```bash
VBoxManage createvm --name "k8s-worker" --ostype Debian_64 --basefolder /vm --register
VBoxManage modifyvm "k8s-worker" --firmware bios --boot1 dvd --boot2 disk --boot3 none --boot4 none
VBoxManage modifyvm "k8s-worker" --memory 10240 --cpus 5 --audio-driver none --graphicscontroller vmsvga --vram 16
VBoxManage createhd --filename /vm/k8s-worker/k8s-worker.vdi --size 40000 --variant Standard
VBoxManage storagectl "k8s-worker" --name "SATA Controller" --add sata --controller IntelAhci
VBoxManage storageattach "k8s-worker" --storagectl "SATA Controller" --port 0 --device 0 --type hdd --medium /vm/k8s-worker/k8s-worker.vdi --nonrotational on --hotpluggable on
VBoxManage modifyvm "k8s-worker" --macaddress1 080027923B5C
VBoxManage modifyvm "k8s-worker" --nic1 bridged --bridgeadapter1 enp5s0
VBoxManage storagectl "k8s-worker" --name "IDE Controller" --add ide
VBoxManage storageattach "k8s-worker" --storagectl "IDE Controller" --port 0 --device 0 --type dvddrive --medium "$OUT_ISO"
```

Start the VM:

```bash
VBoxManage startvm "k8s-worker" --type headless
```

### VM Specifications: k8s-worker
- **Memory:** 10240 MB (10 GB)
- **CPUs:** 5
- **Storage:** 40 GB VDI
- **Network:** Bridged adapter (enp5s0)
- **MAC Address:** 08:00:27:92:3B:5C

## 6. Create VM: kali-linux

Create and configure the Kali Linux VM:

```bash
VBoxManage createvm --name "kali-linux" --ostype Debian_64 --basefolder /vm --register
VBoxManage modifyvm "kali-linux" --firmware bios --boot1 dvd --boot2 disk --boot3 none --boot4 none
VBoxManage modifyvm "kali-linux" --memory 4096 --cpus 2 --audio-driver none --graphicscontroller vmsvga --vram 64
VBoxManage createhd --filename /vm/kali-linux/kali-linux.vdi --size 40000 --variant Standard
VBoxManage storagectl "kali-linux" --name "SATA Controller" --add sata --controller IntelAhci
VBoxManage storageattach "kali-linux" --storagectl "SATA Controller" --port 0 --device 0 --type hdd --medium /vm/kali-linux/kali-linux.vdi --nonrotational on --hotpluggable on
VBoxManage modifyvm "kali-linux" --macaddress1 0800279E476B
VBoxManage modifyvm "kali-linux" --nic1 bridged --bridgeadapter1 enp5s0
VBoxManage storagectl "kali-linux" --name "IDE Controller" --add ide
VBoxManage storageattach "kali-linux" --storagectl "IDE Controller" --port 0 --device 0 --type dvddrive --medium /vm/iso/kali-linux-2025.2-installer-amd64.iso
```

Start the VM:

```bash
VBoxManage startvm "kali-linux" --type headless
```

### VM Specifications: kali-linux
- **Memory:** 4096 MB (4 GB)
- **CPUs:** 2
- **Storage:** 40 GB VDI
- **Network:** Bridged adapter (enp5s0)
- **MAC Address:** 08:00:27:9E:47:6B
- **ISO:** Uses official Kali Linux installer ISO

## 7. Useful Checks and Management Commands

### Verify Firmware Mode
Check that the VM is using BIOS firmware (should show "BIOS"):

```bash
VBoxManage showvminfo "k8s-master" | grep -i firmware
# Output: Firmware: BIOS
```

### Change Firmware to UEFI
If you want to use GRUB flow instead:

```bash
VBoxManage modifyvm "k8s-master" --firmware efi
```

### Set Boot Order Explicitly
Ensure DVD boots first, then disk:

```bash
VBoxManage modifyvm "k8s-master" --boot1 dvd --boot2 disk --boot3 none --boot4 none
```

### List All VMs
```bash
VBoxManage list vms
```

### Check VM Status
```bash
VBoxManage list runningvms
```

### Stop a VM
```bash
# Graceful shutdown
VBoxManage controlvm "vm-name" acpipowerbutton

# Force power off
VBoxManage controlvm "vm-name" poweroff
```

### Get VM Information
```bash
VBoxManage showvminfo "vm-name"
```

## 8. VM Summary Table

| VM Name | Memory | CPUs | Storage | MAC Address | Purpose |
|---------|--------|------|---------|-------------|---------|
| tools | 6 GB | 3 | 35 GB | 08:00:27:27:62:D6 | General tools and utilities |
| k8s-master | 10 GB | 5 | 40 GB | 08:00:27:9E:47:3A | Kubernetes master node |
| k8s-worker | 10 GB | 5 | 40 GB | 08:00:27:92:3B:5C | Kubernetes worker node |
| kali-linux | 4 GB | 2 | 40 GB | 08:00:27:9E:47:6B | Security testing |

## 9. FAQ / Troubleshooting

### VM shows language selection
**Cause:** The kernel cmdline didn't get the preseed params.
**Solution:** 
- Ensure you booted in the intended firmware (BIOS vs UEFI)
- Verify that the ISO contains the edited `txt.cfg`/`grub.cfg`
- In BIOS, press Tab on Install to inspect the cmdline

### Kernel panic "unknown-block(0,0)"
**Cause:** The ISOLINUX append line was split.
**Solution:** Ensure the append line is a single line and includes `initrd=/install.amd/initrd.gz ---`

### Want GUI instead of headless
**Solution:** Remove `--type headless` when starting the VM:
```bash
VBoxManage startvm "vm-name"
```

### Network adapter not found
**Solution:** Replace `enp5s0` with your actual network interface name:
```bash
# List available network interfaces
ip link show

# Update the bridgeadapter parameter
VBoxManage modifyvm "vm-name" --bridgeadapter1 your-interface-name
```

### VM won't start
**Solution:** Check VirtualBox logs and ensure:
- Sufficient system resources available
- VirtualBox kernel modules loaded
- No conflicting virtualization software running

---

## Quick Start Script

Create all VMs at once with this script:

```bash
#!/bin/bash

OUT_ISO="/vm/iso/debian-13.0.0-amd64-netinst-custom.iso"

# Create tools VM
echo "Creating tools VM..."
VBoxManage createvm --name "tools" --ostype Debian_64 --basefolder /vm --register
# ... (add all tools commands here)

# Create k8s-master VM
echo "Creating k8s-master VM..."
VBoxManage createvm --name "k8s-master" --ostype Debian_64 --basefolder /vm --register
# ... (add all k8s-master commands here)

# Create k8s-worker VM
echo "Creating k8s-worker VM..."
VBoxManage createvm --name "k8s-worker" --ostype Debian_64 --basefolder /vm --register
# ... (add all k8s-worker commands here)

echo "All VMs created successfully!"
```

This setup provides a complete lab environment for Kubernetes development and security testing.