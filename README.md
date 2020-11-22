# VFIO Arch Guide

:negative_squared_cross_mark: [**Create an issue**](https://github.com/qdm12/VFIO-Arch-Guide/issues/new)

Please create an issue if you find any problem, I'll be happy to help and fix this document.

## Purpose

I have a self built Arch Linux server running all the time, running mostly Docker containers.

I wanted to setup a Windows 10 virtual machine with GPU passthrough such that:

- I don't have to run another Windows 10 machine to save on Hardware costs and electricity
- Friends without a gaming computer would be able to play on my server
- I could play on my server remotely (= no need to carry a mini ITX build at airports)
- It would be isolated from the rest of the server

**This guide will show you step by step how to setup such virtual machine on an Arch Linux server with precise commands and examples.**

For reference, my configuration is:

- Host OS: `Linux zenarch 5.7.7-arch1-1 #1 SMP PREEMPT Wed, 01 Jul 2020 14:53:16 +0000 x86_64 GNU/Linux`
- CPU: AMD Ryzen 2600x
- Motherboard: ASUS Strix X470-I
- GPU: MSI GTX 1660 Super
- RAM: 32GB

## Pre-requisites

- A motherboard supporting UEFI (most modern motherboards do)
- A graphics card VBIOS supporting UEFI (most modern cards do)
- Access your server through SSH
- Run as `root`, no time to waste `sudo`ing everything

## Motherboard UEFI configuration

- Virtualization enabled (a.k.a. *VT-d* or *AMD-v* or *SVM mode*)
- IOMMU enabled (for me, it was in *Advanced/AMD CBS/NBIO Common Options/NB Configuration*)
- Any kind of CSM **completely disabled**

## Bootloader and IOMMU

The following is for GRUB, although similar instructions should apply to
systemd-boot (i.e. add `amd_iommu=on iommu=pt` to the `options` line in your
boot entry file).

1. Modify `/etc/default/grub` changing:

    ```sh
    GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"
    ```

    to

    ```sh
    GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet amd_iommu=on iommu=pt video=efifb:off"
    ```

   **If you have an Intel CPU**, use `intel_iommu` instead of `amd_iommu`.

1. Rebuild the Grub config

    ```sh
    grub-mkconfig -o /boot/grub/grub.cfg
    ```

1. Reboot

    ```sh
    poweroff --reboot
    ```

1. Check the Linux kernel ring buffer

      ```sh
      dmesg | grep -i -e DMAR -e IOMMU
      ```

    You should see something similar to:

    ```log
    [    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-linux root=UUID=d086663a-xxxxx-xxx rw loglevel=3 quiet amd_iommu=on iommu=pt
    [    0.000000] Kernel command line: BOOT_IMAGE=/boot/vmlinuz-linux root=UUID=d086663a-xxxx-xxx rw loglevel=3 quiet amd
    iommu=on iommu=pt
    [    0.173349] iommu: Default domain type: Passthrough (set via kernel command line)
    [    0.282985] pci 0000:00:00.2: AMD-Vi: IOMMU performance counters supported
    [    0.283054] pci 0000:00:01.0: Adding to iommu group 0
    [    0.283071] pci 0000:00:01.1: Adding to iommu group 1
    [    0.283086] pci 0000:00:01.3: Adding to iommu group 2
    ...
    ...
    ...
    [    0.283703] pci 0000:0a:00.3: Adding to iommu group 21
    [    0.283938] pci 0000:00:00.2: AMD-Vi: Found IOMMU cap 0x40
    [    0.284262] perf/amd_iommu: Detected AMD IOMMU #0 (2 banks, 4 counters/bank).
    [    0.292351] AMD-Vi: AMD IOMMUv2 driver by Joerg Roedel <jroedel@suse.de>
    ```

## Find your IOMMU group

Find the group your GPU belongs to.

1. Save a file `script.sh` with content

    ```sh
    #!/bin/bash
    shopt -s nullglob
    for g in /sys/kernel/iommu_groups/*; do
        echo "IOMMU Group ${g##*/}:"
        for d in $g/devices/*; do
            echo -e "\t$(lspci -nns ${d##*/})"
        done;
    done;
    ```

1. Run it

    ```sh
    chmod +x script.sh
    ./script.sh
    ```

1. You should obtain something similar to

    ```log
    IOMMU Group 0:
            00:01.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Family 17h (Models 00h-1fh) PCIe Dummy Host Bridge [1022:1452]
    ...
    ...
    ...
    IOMMU Group 15:
            08:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU116 [GeForce GTX 1660 SUPER] [10de:21c4] (rev a1)
            08:00.1 Audio device [0403]: NVIDIA Corporation TU116 High Definition Audio Controller [10de:1aeb] (rev a1)
            08:00.2 USB controller [0c03]: NVIDIA Corporation Device [10de:1aec] (rev a1)
            08:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU116 [GeForce GTX 1650 SUPER] [10de:1aed] (rev a1)
    ...
    ...
    ...
    IOMMU Group 9:
            00:08.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Family 17h (Models 00h-1fh) PCIe Dummy Host Bridge [1022:1452]
    ```

    Here the group we want is group `15`. If you have something unusual, see [this](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Gotchas).

## Binding devices to VFIO-PCI

We need to bind all devices of the IOMMU Group `15` to the VFIO PCI driver at boot.

1. Edit `/etc/default/grub` changing:

    ```sh
    GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet amd_iommu=on iommu=pt video=efifb:off"
    ```

    to (replacing the IDs from device IDs you obtained above)

    ```sh
    GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet amd_iommu=on iommu=pt video=efifb:off vfio-pci.ids=10de:21c4,10de:1aeb,10de:1aec,10de:1aed"
    ```

1. Rebuild the Grub config

    ```sh
    grub-mkconfig -o /boot/grub/grub.cfg
    ```

## Kernel modules

1. Modify `/etc/mkinitcpio.conf` and add `vfio_pci vfio vfio_iommu_type1 vfio_virqfd` to your `MODULES` array.

    For example, change from

    ```sh
    MODULES=()
    ```

    to

    ```sh
    MODULES=(vfio_pci vfio vfio_iommu_type1 vfio_virqfd)
    ```

1. Regenerate the initramfs with

    ```sh
    mkinitcpio -p linux
    ```

## Reboot and check

1. Reboot

    ```sh
    poweroff --reboot
    ```

1. Check the VFIO kernel module got loaded:

    ```sh
    dmesg | grep -i vfio
    ```

   Should output

    ```log
    [    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-linux root=UUID=d086663a-9179-48c6-9946-1035da860829 rw loglevel=3 quiet amd_iommu=on iommu=pt vfio-pci.ids=10de:21c4,10de:1aeb,10de:1aec,10de:1aed
    [    0.000000] Kernel command line: BOOT_IMAGE=/boot/vmlinuz-linux root=UUID=d086663a-9179-48c6-9946-1035da860829 rw loglevel=3 quiet amd_iommu=on iommu=pt vfio-pci.ids=10de:21c4,10de:1aeb,10de:1aec,10de:1aed
    [    3.396953] VFIO - User Level meta-driver version: 0.3
    [    3.400366] vfio-pci 0000:09:00.0: vgaarb: changed VGA decodes: olddecodes=io+mem,decodes=io+mem:owns=io+mem
    [    3.416692] vfio_pci: add [10de:21c4[ffffffff:ffffffff]] class 0x000000/00000000
    [    3.433353] vfio_pci: add [10de:1aeb[ffffffff:ffffffff]] class 0x000000/00000000
    [    3.450019] vfio_pci: add [10de:1aec[ffffffff:ffffffff]] class 0x000000/00000000
    [    3.466953] vfio_pci: add [10de:1aed[ffffffff:ffffffff]] class 0x000000/00000000
    ```

    You can check for example that the device ID `10de:21c4` got added with:

    ```sh
    lspci -nnk -d 10de:21c4
    ```

    Showing

    ```log
    09:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU116 [GeForce GTX 1660 SUPER] [10de:21c4] (rev a1)
            Subsystem: Micro-Star International Co., Ltd. [MSI] Device [1462:c75a]
            Kernel driver in use: vfio-pci
            Kernel modules: nouveau
    ```

## Virtual machine requirements

1. Install necessary packages

    ```sh
    pacman -Sy qemu edk2-ovmf ebtables dnsmasq
    ```

1. Setup `yay` to install AUR packages:

    ```sh
    # Setting up non root user for yay
    useradd -m nonroot
    mkdir -p /etc/sudoers.d
    echo nonroot ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/nonroot
    chmod 0440 /etc/sudoers.d/nonroot

    # Installing yay
    originPath="$(pwd)"
    mkdir /tmp/yay
    cd /tmp/yay
    git clone --single-branch --depth 1 https://aur.archlinux.org/yay.git .
    pacman -Sy -q --needed --noconfirm go
    mkdir -p /home/nonroot/.cache
    chown -R nonroot /tmp/yay /.cache
    sudo -u nonroot makepkg
    pacman -R --noconfirm go
    pacman -U --noconfirm yay*.tar.zst
    cd "$originPath"
    rm -r /tmp/yay /home/nonroot/.cache
    ```

1. Install `downgrade` so we can install the versions we want for some of the packages:

    ```sh
    su nonroot -c "yay -Sy --noconfirm downgrade"
    ```

1. Install `libvirt 6.8.0-3` interactively using `downgrade`:

    ```sh
    downgrade libvirt
    ```

1. Enable and start the libvirtd service and its logging component virtlogd.socket

    ```sh
    systemctl enable --now libvirtd
    systemctl enable --now virtlogd.socket
    ```

1. Activate the default virtual network

    ```sh
    virsh net-start default
    ```

## Virtual machine management GUI

To configure the guest OS, we will use the graphical interface `virt-manager`.

1. Install `virt-manager 3.1.0-2` (or more) interactively using `downgrade`:

    ```sh
    downgrade libvirt
    ```

1. Install `xorg-xauths` to have access to the GUI through SSH.

    ```sh
    pacman -Sy xorg-xauths
    ```

1. On your server, modify `/etc/ssh/sshd_config`, add the line `X11Forwarding yes` and restart sshd:

    ```sh
    systemctl restart sshd
    ```

1. On your computer/client, you need to specify the `-X` flag to your SSH connect command to forward X11.
If you are on Windows, use [MobaXterm](https://mobaxterm.mobatek.net/download.html) in order to have X11 support.
1. Once you are logged in your server over SSH with X11:

    ```sh
    virt-manager --no-fork
    ```

    And wait a few seconds for the GUI to show.

## Download the Windows 10 iso

1. Go to [https://www.microsoft.com/en-us/software-download/windows10ISO](https://www.microsoft.com/en-us/software-download/windows10ISO)
1. Select the Windows 10 edition, language and architecture you want to download the iso file.
1. Rename it to `Windows10.iso` and place it somewhere on your server.

## Virtual machine configuration

### Creating the virtual machine

1. Click on *File*, *New Virtual Machine*
1. Select *Local Install media (ISO image or CDROM)* and click *Forward*
1. Browse to your Windows 10 iso file. You may need to create a file directory pool in order to browse to it.
1. Enter manually the operating system being installed as `Microsoft Windows 10`
1. Click on *Forward*
1. Set the memory and CPU to what you desire, ideally 8GB and 4 CPUs at least. Click on *Forward*
1. Either create a disk image (simpler) or select a custom storage.
In my case, I added a filesystem directory pool at `/dev` and picked my blank new ssd `/dev/sdd` from it as its custom storage. Click on *Forward*.
1. Set the name to `win10` for consistency with the following instructions.
1. Tick *Customize configuration before install*.
1. Expand the *Network selection* to avoid `virt-manager` to crash 👍
1. Click *Finish*
1. In the *Overview* section, change the Firmware to UEFI. In my case it was `UEFI x86_64:/usr/share/edk2-ovmf/x64/OVMF_CODE.fd`. Click *Apply*.
1. In the *CPUs* section, untick *Copy host CPU configuration* and choose the `host-passthrough` for the *Model*. Click *Apply*.
1. In the *Boot Options* section, tick the *Enable boot menu* and tick the *SATA CDROM 1* so that it can boot from the windows 10 iso file. Click *Apply*.
1. Click on **Begin Installation**. A window will open and you can follow Windows installation steps interactively.

### VFIO Windows setup

Once you are done setting up Windows 10, in the virtual machine:

1. Download the [Virtio drivers for Windows 10](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.173-7/virtio-win-gt-x64.msi) and install it.
1. Download the Nvidia/AMD drivers for your graphics card but **do not** install it yet.
1. Download and install [Parsec](https://parsecgaming.com/).
1. Log in to your Parsec account.
1. **Configure Parsec to start with Windows**.
1. Shut down the virtual machine.
1. Quit `virt-manager`.

### Hide the VM

We will edit the Windows 10 libvirt xml file to hide away we are running in a virtual machine so graphics drivers won't detect that we are running in a virtual machine.

Open your editor with

```sh
virsh edit win10
```

All the following modifications have to be done in the block

```xml
<features>
  ...
</features>
```

1. Add the following in the `<features>` XML block

    ```xml
    <ioapic driver='kvm'/>
    ```

1. Add the following in the `<features>` XML block

    ```xml
    <kvm>
      <hidden state='on'/>
    </kvm>
    ```

1. Make the Hyperv vendor ID random, by adding to the `<hyperv>` XML block:

    ```xml
    <vendor_id state='on' value='123412341234'/>
    ```

### Add the Graphics card

1. Start `virt-manager`

  ```sh
  virt-manager --no-fork
  ```

1. Double click on *win10*
1. Click on the 💡 icon to show hardware details
1. For each of the PCI devices in the IOMMU group you want to add:
    1. Click on *Add Hardware*
    1. Click on *PCI Host Device*
    1. Select the device and click on *Finish* (example: `0000:09:00:0 NVIDIA Corporation TU116 [GeForce GTX 1660 Super]`).
1. Plug a screen on your graphics card 📺 🔌
1. Start the virtual machine ⏯️
1. Although nothing moves on your screen, your devices control the virtual machine, you can see it on the physical screen you plugged. Don't worry that will just be to install the graphics drivers 😖 !
1. Install the Nvidia/AMD drivers now
1. Shut down the virtual machine ⏹️

### Remove unneeded devices

In virt-manager, remove the following devices that we no longer need:

- SATA CDROM1
- Display Spice
- Channel spice
- Video QXL
- Tablet
- Serial 1
- USB Redirector 1
- USB Redirector 2
- Virtio Serial Controller

## Final steps

1. Boot the virtual machine
1. Install Parsec on your client device, log in and connect to your virtual machine
1. Profit 🎉 🎆 🎉

## References

- [Arch Linux Wiki on PCI Passthrough via OVMF](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF)
- [blog.zerosector.io article on KVM QEMU Windows 10 GPU Passthrough](https://blog.zerosector.io/2018/07/28/kvm-qemu-windows-10-gpu-passthrough/)
- [Vanities' Github on GPU passthrough Arch Linux to Windows 10](https://github.com/vanities/GPU-Passthrough-Arch-Linux-to-Windows10)

## Further work

Automate the setup with `virsh` to avoid relying on the `virt-manager` GUI. For example:

```sh
virt-install -n gaming \
  --description "Gaming VM with VFIO" \
  --os-type=Windows \
  --os-variant= \
  --ram=8096 \
  --vcpus=4 \
  --disk path=/var/lib/libvirt/images/myRHELVM1.img,bus=virtio,size=250 \
  --cdrom=.iso \
  --dry-run
  ...
```
