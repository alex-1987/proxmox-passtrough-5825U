
# Proxmox Passthrough Guide
================================

## Introduction
---------------

Thanks to the contributions of [isc30](https://github.com/isc30/ryzen-7000-series-proxmox?tab=readme-ov-file) and my own tests, I'm happy to share this guide on how to set up Proxmox passthrough.

## Non-Subscription Updates
---------------------------

To get Proxmox updates without a subscription, run the following command:
```bash
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/misc/post-pve-install.sh)"
```

## Install CPU Microcode Packages
--------------------------------

Install the CPU microcode packages by running:
```bash
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/misc/microcode.sh)"
```

## Identify AMD/ATI Devices
---------------------------

Run the following command to identify AMD/ATI devices:
```bash
lspci -nn | grep -e 'AMD/ATI'
```

### Output
----------

The output should look something like this:
```
05:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Barcelo [1002:15e7] (rev c1)
05:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Renoir Radeon High Definition Audio Controller [1002:1637]
```

## Configure GRUB
-----------------

Add `iommu=pt` to GRUB by running:
```bash
sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="quiet"/GRUB_CMDLINE_LINUX_DEFAULT="quiet iommu=pt"/g' /etc/default/grub
update-grub
```

## Configure VFIO
-----------------

Configure VFIO by adding the following lines to `/etc/modprobe.d/vfio.conf`:
```bash
echo "vfio" >> /etc/modules
echo "vfio_iommu_type1" >> /etc/modules
echo "vfio_pci" >> /etc/modules
echo "vfio_virqfd" >> /etc/modules
echo "options vfio-pci ids=1002:15e7,1002:1637" >> /etc/modprobe.d/vfio.conf

echo "softdep amdgpu pre: vfio-pci" >> /etc/modprobe.d/vfio.conf
echo "softdep snd_hda_intel pre: vfio-pci" >> /etc/modprobe.d/vfio.conf

echo "blacklist amdgpu" >> /etc/modprobe.d/pve-blacklist.conf
echo "blacklist snd_hda_intel" >> /etc/modprobe.d/pve-blacklist.conf
```

## Update Initramfs
--------------------

Update initramfs by running:
```bash
update-initramfs -u -k all
shutdown -r now
```

## Create BIOS .bin File
-------------------------

### Create vbios.c File


Create a `vbios.c` file with the following contents:

<details>
  <summary>Expand vbios.c</summary>
  
  ```
  #include <stdint.h>
  #include <stdio.h>
  #include <stdlib.h>
  
  typedef uint32_t ULONG;
  typedef uint8_t UCHAR;
  typedef uint16_t USHORT;
  
  typedef struct {
      ULONG Signature;
      ULONG TableLength; // Length
      UCHAR Revision;
      UCHAR Checksum;
      UCHAR OemId[6];
      UCHAR OemTableId[8]; // UINT64  OemTableId;
      ULONG OemRevision;
      ULONG CreatorId;
      ULONG CreatorRevision;
  } AMD_ACPI_DESCRIPTION_HEADER;
  
  typedef struct {
      AMD_ACPI_DESCRIPTION_HEADER SHeader;
      UCHAR TableUUID[16]; // 0x24
      ULONG VBIOSImageOffset; // 0x34. Offset to the first GOP_VBIOS_CONTENT block from the beginning of the stucture.
      ULONG Lib1ImageOffset; // 0x38. Offset to the first GOP_LIB1_CONTENT block from the beginning of the stucture.
      ULONG Reserved[4]; // 0x3C
  } UEFI_ACPI_VFCT;
  
  typedef struct {
      ULONG PCIBus; // 0x4C
      ULONG PCIDevice; // 0x50
      ULONG PCIFunction; // 0x54
      USHORT VendorID; // 0x58
      USHORT DeviceID; // 0x5A
      USHORT SSVID; // 0x5C
      USHORT SSID; // 0x5E
      ULONG Revision; // 0x60
      ULONG ImageLength; // 0x64
  } VFCT_IMAGE_HEADER;
  
  typedef struct {
      VFCT_IMAGE_HEADER VbiosHeader;
      UCHAR VbiosContent[1];
  } GOP_VBIOS_CONTENT;
  
  int main(int argc, char** argv)
  {
      FILE* fp_vfct;
      FILE* fp_vbios;
      UEFI_ACPI_VFCT* pvfct;
      char vbios_name[0x400];
  
      if (!(fp_vfct = fopen("/sys/firmware/acpi/tables/VFCT", "r"))) {
          perror(argv[0]);
          return -1;
      }
  
      if (!(pvfct = malloc(sizeof(UEFI_ACPI_VFCT)))) {
          perror(argv[0]);
          return -1;
      }
  
      if (sizeof(UEFI_ACPI_VFCT) != fread(pvfct, 1, sizeof(UEFI_ACPI_VFCT), fp_vfct)) {
          fprintf(stderr, "%s: failed to read VFCT header!\n", argv[0]);
          return -1;
      }
  
      ULONG offset = pvfct->VBIOSImageOffset;
      ULONG tbl_size = pvfct->SHeader.TableLength;
  
      if (!(pvfct = realloc(pvfct, tbl_size))) {
          perror(argv[0]);
          return -1;
      }
  
      if (tbl_size - sizeof(UEFI_ACPI_VFCT) != fread(pvfct + 1, 1, tbl_size - sizeof(UEFI_ACPI_VFCT), fp_vfct)) {
          fprintf(stderr, "%s: failed to read VFCT body!\n", argv[0]);
          return -1;
      }
  
      fclose(fp_vfct);
  
      while (offset < tbl_size) {
          GOP_VBIOS_CONTENT* vbios = (GOP_VBIOS_CONTENT*)((char*)pvfct + offset);
          VFCT_IMAGE_HEADER* vhdr = &vbios->VbiosHeader;
  
          if (!vhdr->ImageLength)
              break;
  
          snprintf(vbios_name, sizeof(vbios_name), "vbios_%x_%x.bin", vhdr->VendorID, vhdr->DeviceID);
  
          if (!(fp_vbios = fopen(vbios_name, "wb"))) {
              perror(argv[0]);
              return -1;
          }
  
          if (vhdr->ImageLength != fwrite(&vbios->VbiosContent, 1, vhdr->ImageLength, fp_vbios)) {
              fprintf(stderr, "%s: failed to dump vbios %x:%x\n", argv[0], vhdr->VendorID, vhdr->DeviceID);
              return -1;
          }
  
          fclose(fp_vbios);
  
          printf("dump vbios %x:%x to %s\n", vhdr->VendorID, vhdr->DeviceID, vbios_name);
  
          offset += sizeof(VFCT_IMAGE_HEADER);
          offset += vhdr->ImageLength;
      }
  
      return 0;
  }
  ```
</details>

### Compile and Run the Script
  
Compile the script using `gcc` and run it:
```bash
gcc vbios.c -o vbios
./vbios
```


### Download AMDGopDriver-5825U.rom

Download the `AMDGopDriver-5825U.rom` file from [alex-1987/proxmox-passtrough-5825U](https://github.com/alex-1987/proxmox-passtrough-5825U/blob/main/AMDGopDriver-5825U.rom).

### Rename and Move Files

Rename and move the files to the correct locations:
```bash
mv vbios_*.bin vbios_5825U.bin
mv vbios_5825U.bin /usr/share/kvm/
mv AMDGopDriver-5825U.rom /usr/share/kvm/
```

## Update PVE Config
---------------------

Update your PVE config file (`/etc/pve/qemu-server/1010.conf`) with the following lines:
```javascript
hostpci0: 0000:05:00.0,romfile=vbios_5825U.bin,x-vga=1
hostpci1: 0000:05:00.1,romfile=AMDGopDriver-5825U.rom
```

## Check IGPU works
---------------------------

Check if the CPU is in the dev list by running:
```bash
ls /dev/dri/
```

The output should look something like this:
```
by-path  card0  renderD128
```

Check with lspci the driver in use:
```bash
 lspci -nnk | grep AMD -A 3
```

output:
```
06:10.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Barcelo [1002:15e7] (rev c1)
        Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Barcelo [1002:0123]
        Kernel driver in use: amdgpu
        Kernel modules: amdgpu
06:11.0 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Renoir Radeon High Definition Audio Controller [1002:1637]
        Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Renoir Radeon High Definition Audio Controller [1002:1637]
        Kernel driver in use: snd_hda_intel
        Kernel modules: snd_hda_intel
06:12.0 Ethernet controller [0200]: Red Hat, Inc. Virtio network device [1af4:1000]
```

## VM Config
-------------

Here is an example VM config file (`/etc/pve/qemu-server/1010.conf`):
```javascript
agent: 1
bios: ovmf
boot: order=scsi0;ide2;net0
cores: 10
cpu: x86-64-v2-AES
efidisk0: local-lvm:vm-1010-disk-0,efitype=4m,pre-enrolled-keys=1,size=4M
hostpci0: 0000:05:00.0,romfile=vbios_5825U.bin,x-vga=1
hostpci1: 0000:05:00.1,romfile=AMDGopDriver-5825U.rom
ide2: none,media=cdrom
machine: q35
memory: 16384
meta: creation-qemu=9.0.2,ctime=1736754641
name: Docker-Server
net0: virtio=BC:24:11:A2:E5:CD,bridge=vmbr1,firewall=1
numa: 0
ostype: l26
scsi0: data:vm-1010-disk-1,iothread=1,size=32G
scsi1: data:vm-1010-disk-6,size=5000G
scsi2: data:vm-1010-disk-3,iothread=1,size=100G
scsi3: data:vm-1010-disk-4,iothread=1,size=1000G
scsi4: data:vm-1010-disk-5,iothread=1,size=1T
scsihw: virtio-scsi-single
serial0: socket
smbios1: uuid=5b04eac1-df70-4d31-b278-7208b2d76055
sockets: 1
unused0: data:vm-1010-disk-0
unused1: data:vm-1010-disk-2
usb0: host=1-4
vga: none
vmgenid: 4a8a252e-4fc5-460b-a436-443fd12d0274
```
```
