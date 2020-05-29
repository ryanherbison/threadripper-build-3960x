threadripper-build-3960x

Workstation Build Specifications
--------------------------------
Motherboard: Asrock Creator TRX40 bios 1.6

Ram: 64gb 4x16gb CMU32GX4M2C3000C15 OC to 3200

Processor: 3960x Threadripper

Boot Graphics: PCIe - Slot #1 - Quadro 2000

Graphics #2: AMD RX580

Graphics #3: Asus NVidia 1080

Displays (4)

Primary OS - Gentoo

VM #1 - Windows 10 on 1080

VM #2 - Gento on RX580

GPU Passthru

Bios Settings: There are three or four things that need to be turned on in the Bios. Will make a list, at least one of them needs to be enabled instead of just set to auto.

Kernel Settings: amd_iommu=on iommu=pt pci=noaer

Notes: pci-noaer fixed issue with AMD RX580 failing to start with Qemu

Additional Kernel Setting: pcie_no_flr=1022:149C,1022:1487,1022:1048C
This line requires a kernel patch to quirks.cOVMF 
The built-in audio and two of the usb controllers have reset issues, they advertise the ability to reset but fail to implement when called.

Kernel Patch:
Add the following three lines to the function:

Issues encountered:
Setup initial VM with Virt-Manager and then check the virt-manager logs for the actual command that is executed. You can use
that command to construct a functional qemu config that can work from a bash script.

I had some problems trying to use my previous Windows10 config from an intel system. Solution was to recreate an initial qemu config and tweak that. Turns out the OVMF bios I was using was out of date but included in Gentoo/KVM/QEMU. A lot of changes have been made in QEMU so your old configs probably will have issues.

modprobe.d configs

blacklist.conf
------------------
blacklist atlantic
blacklist nvidia

kvm-nested.conf
---------------
options kvm-intel nested=1
options kvm-intel enable_shadow_vmcs=1
options kvm-intel enable_apicv=1
options kvm-intel ept=1

vfio-pci.conf
-------------
softdep amdgpu pre: vfio-pci
softdep radeon pre: vfio-pci
softdep nvidia pre: vfio-pci
softdep nvidia* pre: vfio-pci
softdep i2c_nvidia_gpu pre: vfio-pci
softdep nvidia-gpu pre: vfio-pci
softdep xhci_hcd pre: vfio-pci

options vfio-pci ids=10de:1b80,10de:10f0,1d6a:07b1,1022:1487,1b21:3242,10ec:8168,1002:aaf0,1002:67df,1022:148c disable_vga=1
options kvm_amd avic=1 npt=1 nested=1

#10de      GTX1080
#1d6a:07b1 Aquantia NIC
#1022:1487 Onboard Sound (Kernel Patch)
#1b21:3242 USB 3.O Controller Back
#10ec:8168 Realtek Gigabit Add-On Card
#1002:aaf0 Audio on AMD RTX580

QEMU config for Windows
-----------------------
#!/bin/bash

vmname="mywindows10"
if ps -A | grep -q $vmname; then
   echo "$vmname is already running." &
   exit 1

else

qemu-system-x86_64  \
  -name $vmname,process=$vmname \
  -machine pc-q35-4.2,accel=kvm,usb=off,vmport=off,dump-guest-core=off,kernel_irqchip=on \
  -cpu EPYC-IBPB,tsc-deadline=on,hypervisor=on,tsc-adjust=on,clwb=on,umip=on,stibp=on,arch-capabilities=on,ssbd=on,xsaves=on,cmp-legacy=on,perfctr-core=on,clzero=on,wbnoinvd=on,amd-ssbd=on,virt-ssbd=on,rdctl-no=on,skip-l1dfl-vmentry=on,mds-no=on,monitor=off,x2apic=off,hv-time,hv-relaxed,hv-vapic,hv-spinlocks=0x1fff,hv_vendor_id=null,kvm=off,topoext \
  -smp 16,sockets=1,cores=16,threads=1 \
  -enable-kvm \
  -m 26G \
  -mem-path /dev/hugepages \
  -rtc base=localtime,driftfix=slew \
  -vga none \
  -nographic \
  -serial none \
  -parallel none \
  -device vfio-pci,host=0000:44:00.0 \
  -device vfio-pci,host=0000:43:00.0 \
  -device vfio-pci,host=0000:21:00.0,multifunction=on \
  -device vfio-pci,host=0000:21:00.1 \
  -drive if=pflash,format=raw,readonly,file=/usr/share/edk2-ovmf/OVMF_CODE.fd,unit=0 \
  -drive if=pflash,format=raw,file=/usr/share/edk2-ovmf/OVMF_VARS.fd,unit=1 \
  -boot order=dc \
  -device virtio-scsi-pci,id=scsi \
  -drive id=disk0,if=virtio,format=raw,cache=none,file=/dev/disk/by-id/wwn-0x5002538d41c67120,aio=native \
  -drive id=disk1,if=virtio,format=raw,file=/mnt/bigdata/WindowsVirt/win10-secondary.img \
  -drive id=disk2,if=virtio,format=raw,file=/root/images/windows-10-fast.img &

Script for showing iommu groups well formatted - usb/storage breakdown
----------------------------------------------------------------------
#!/bin/bash

useColors=true
usePager=true

usage() {
	echo "\
Usage: $(basename $0) [OPTIONS]
Shows information about IOMMU groups relevant for working with PCI-passthrough

  -c -C 	enables/disables colored output, respectively
  -p -P 	enables/disables pager (less), respectively

  -h 		display this help message"
}

color() {
	if ! $useColors; then
		cat
		return
	fi

	rset=$'\E[0m'
	case "$1" in
		black) colr=$'\E[22;30m' ;;
		red) colr=$'\E[22;31m' ;;
		green) colr=$'\E[22;32m' ;;
		yellow) colr=$'\E[22;33m' ;;
		blue) colr=$'\E[22;34m' ;;
		magenta) colr=$'\E[22;35m' ;;
		cyan) colr=$'\E[22;36m' ;;
		white) colr=$'\E[22;37m' ;;
		intenseBlack) colr=$'\E[01;30m' ;;
		intenseRed) colr=$'\E[01;31m' ;;
		intenseGreen) colr=$'\E[01;32m' ;;
		intenseYellow) colr=$'\E[01;33m' ;;
		intenseBlue) colr=$'\E[01;34m' ;;
		intenseMagenta) colr=$'\E[01;35m' ;;
		intenseCyan) colr=$'\E[01;36m' ;;
		intenseWhite) colr=$'\E[01;37m' ;;
	esac

	sed "s/^/$colr/;s/\$/$rset/"
}

indent() {
	sed 's/^/\t/'
}

pager() {
	if $usePager; then
		less -SR
	else
		cat
	fi
}

while getopts cCpPh opt; do
	case $opt in
		c)
			useColors=true
			;;
		C)
			useColors=false
			;;
		p)
			usePager=true
			;;
		P)
			usePager=false
			;;
		h)
			usage
			exit
			;;
	esac
done

iommuGroups=$(find '/sys/kernel/iommu_groups/' -maxdepth 1 -mindepth 1 -type d)

if [ -z "$iommuGroups" ]; then
	echo "No IOMMU groups found. Are you sure IOMMU is enabled?"
	exit
fi

for iommuGroup in $iommuGroups; do
	echo "IOMMU group $(basename "$iommuGroup")" | color red

	for device in $(ls -1 "$iommuGroup/devices/"); do
		devicePath="$iommuGroup/devices/$device/"

		# Print pci device
		lspci -nns "$device" | color blue

		# Print drivers
		driverPath=$(readlink "$devicePath/driver")
		if [ -z "$driverPath" ]; then
			echo "Driver: none"
		else
			echo "Driver: $(basename $driverPath)"
		fi | indent | color cyan

		# Print usb devices
		usbBuses=$(find $devicePath -maxdepth 2 -path '*usb*/busnum')
		for usb in $usbBuses; do
			echo 'Usb bus:' | color cyan
			lsusb -s $(cat "$usb"): | indent | color green
		done | indent

		# Print block devices
		blockDevices=$(find $devicePath -mindepth 5 -maxdepth 5 -name 'block')
		for blockDevice in $blockDevices; do
			echo 'Block device:' | color cyan
			echo "Model: $(cat "$blockDevice/../model")" | indent | color green
			lsblk -no NAME,SIZE,MOUNTPOINT "/dev/$(ls -1 $blockDevice)" | indent | color green
		done | indent
	done | indent
done | pager

Asrock TRX40 Creator iommu groups:
----------------------------------
IOMMU group 55
	45:00.0 Network controller [0280]: Intel Corporation Wi-Fi 6 AX200 [8086:2723] (rev 1a)
IOMMU group 17
	20:02.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 45
	41:00.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Matisse Switch Upstream [1022:57ad]
IOMMU group 35
	40:02.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 7
	00:07.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1484]
IOMMU group 63
	60:02.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 25
	20:08.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1484]
IOMMU group 53
	43:00.0 USB controller [0c03]: ASMedia Technology Inc. Device [1b21:3242]
IOMMU group 15
	03:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Starship USB 3.0 Host Controller [1022:148c]
IOMMU group 43
	40:08.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 71
	61:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Function [1022:148a]
IOMMU group 33
	40:01.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse GPP Bridge [1022:1483]
IOMMU group 5
	00:05.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 61
	4e:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Reserved SPP [1022:1485]
IOMMU group 23
	20:07.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1484]
IOMMU group 51
	42:09.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Matisse PCIe GPP Bridge [1022:57a4]
	48:00.0 SATA controller [0106]: Advanced Micro Devices, Inc. [AMD] FCH SATA Controller [AHCI mode] [1022:7901] (rev 51)
IOMMU group 13
	02:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Function [1022:148a]
IOMMU group 41
	40:07.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 31
	23:00.4 Audio device [0403]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse HD Audio Controller [1022:1487]
IOMMU group 3
	00:03.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 21
	20:05.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 11
	00:18.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship Device 24; Function 0 [1022:1490]
	00:18.1 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship Device 24; Function 1 [1022:1491]
	00:18.2 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship Device 24; Function 2 [1022:1492]
	00:18.3 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship Device 24; Function 3 [1022:1493]
	00:18.4 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship Device 24; Function 4 [1022:1494]
	00:18.5 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship Device 24; Function 5 [1022:1495]
	00:18.6 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship Device 24; Function 6 [1022:1496]
	00:18.7 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship Device 24; Function 7 [1022:1497]
IOMMU group 68
	60:07.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1484]
IOMMU group 1
	00:01.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse GPP Bridge [1022:1483]
IOMMU group 58
	4b:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller [10ec:8168] (rev 15)
IOMMU group 48
	42:03.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Matisse PCIe GPP Bridge [1022:57a3]
IOMMU group 38
	40:03.2 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse GPP Bridge [1022:1483]
IOMMU group 66
	60:05.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 28
	23:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Reserved SPP [1022:1485]
IOMMU group 56
	46:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8125 2.5GbE Controller [10ec:8125] (rev 01)
IOMMU group 18
	20:03.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 46
	42:01.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Matisse PCIe GPP Bridge [1022:57a3]
IOMMU group 36
	40:03.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 8
	00:08.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 64
	60:03.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 26
	21:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP104 [GeForce GTX 1080] [10de:1b80] (rev a1)
	21:00.1 Audio device [0403]: NVIDIA Corporation GP104 High Definition Audio Controller [10de:10f0] (rev a1)
IOMMU group 54
	44:00.0 Ethernet controller [0200]: Aquantia Corp. AQC107 NBase-T/IEEE 802.3bz Ethernet Controller [AQtion] [1d6a:07b1] (rev 02)
IOMMU group 16
	20:01.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 44
	40:08.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1484]
IOMMU group 72
	62:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Reserved SPP [1022:1485]
IOMMU group 34
	40:01.3 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse GPP Bridge [1022:1483]
IOMMU group 6
	00:07.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 62
	60:01.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 24
	20:08.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 52
	42:0a.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Matisse PCIe GPP Bridge [1022:57a4]
	49:00.0 SATA controller [0106]: Advanced Micro Devices, Inc. [AMD] FCH SATA Controller [AHCI mode] [1022:7901] (rev 51)
IOMMU group 14
	03:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Reserved SPP [1022:1485]
IOMMU group 42
	40:07.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1484]
IOMMU group 70
	60:08.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1484]
IOMMU group 32
	40:01.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 4
	00:04.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 60
	4d:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Function [1022:148a]
IOMMU group 22
	20:07.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 50
	42:08.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Matisse PCIe GPP Bridge [1022:57a4]
	47:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Reserved SPP [1022:1485]
	47:00.1 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Matisse USB 3.0 Host Controller [1022:149c]
	47:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Matisse USB 3.0 Host Controller [1022:149c]
IOMMU group 12
	01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GF106GL [Quadro 2000] [10de:0dd8] (rev a1)
	01:00.1 Audio device [0403]: NVIDIA Corporation GF106 High Definition Audio Controller [10de:0be9] (rev a1)
IOMMU group 40
	40:05.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 69
	60:08.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 30
	23:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Starship USB 3.0 Host Controller [1022:148c]
IOMMU group 2
	00:02.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 59
	4c:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere [Radeon RX 470/480/570/570X/580/580X/590] [1002:67df] (rev e7)
	4c:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere HDMI Audio [Radeon RX 470/480 / 570/580/590] [1002:aaf0]
IOMMU group 20
	20:04.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 49
	42:04.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Matisse PCIe GPP Bridge [1022:57a3]
IOMMU group 10
	00:14.0 SMBus [0c05]: Advanced Micro Devices, Inc. [AMD] FCH SMBus Controller [1022:790b] (rev 61)
	00:14.3 ISA bridge [0601]: Advanced Micro Devices, Inc. [AMD] FCH LPC Bridge [1022:790e] (rev 51)
IOMMU group 39
	40:04.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 67
	60:07.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 29
	23:00.1 Encryption controller [1080]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Cryptographic Coprocessor PSPCPP [1022:1486]
IOMMU group 0
	00:01.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 57
	4a:00.0 Non-Volatile memory controller [0108]: Phison Electronics Corporation E16 PCIe4 NVMe Controller [1987:5016] (rev 01)
IOMMU group 19
	20:03.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse GPP Bridge [1022:1483]
IOMMU group 47
	42:02.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Matisse PCIe GPP Bridge [1022:57a3]
IOMMU group 37
	40:03.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse GPP Bridge [1022:1483]
IOMMU group 9
	00:08.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1484]
IOMMU group 65
	60:04.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU group 27
	22:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Function [1022:148a]
