---
layout: post
title: "Virtualization Notes"
date: 2026-08-16
categories: Virtualization
---

As I'm delving in NetBSD, I have encountered that perhaps NetBSD's installer doesn't recognise my NVME disk, also I'm pretty sure the hardware support for my latest Intel wifi cards,
processor (generation of Raptor Lake, which is quite an enhanced version of alder lake), and then my IRISxe integrated graphics card is not on the supported device list.
Its obvious though, NetBSD caters to older devices that needs an operating system that has lower baseline of memory footprint, smaller and cleaner sound server, and support for older obscure hardware.
So, it becomes important to know ins and outs of virtualization techniques on a surface level. As I would be relying on nographics on initial stage, though I could have perfectly spun up a vm and then used ssh to get remote access,
I don't like the idea, because initally I want to understand the overall system layout, go and look through the kernel and its codebase, get more understanding on how the processor support works and what exactly stays same across various generations of Intel CPU's and also 
similarliy with Intel's graphics card, which are in most cases integrated. Then I'm more interested on the sound server and NetBSD's implementation of it at kernel level, which is completely different from Linux ALSA framework.
Next target would be bluetooth obviously, a painfully difficult technology to implement, and atlast graphics stack, getting hardware acceleration and more info on framebuffering mode.
This post will mostly cover what exactly is qemu, then how do kvm and xen differ, also techniques like pcie forwarding of xen hypervisor and its equivalent of kvm.

So qemu stands for quick emulation, its for most part a software level emulator, that can emulate different architecture via TCG (Tiny Code Generator) emulation,more info here:
```txt
https://www.qemu.org/docs/master/devel/index-tcg.html
```
So from upstream docs, whenever qemu encounters a code block of a particular ISA, it converts it to native ISA code block using a dynamic translator, and QEMU's dynamic translator backend is TCG.
QEMU is foremost a userspace process that emulates disk controllers, network cards, GPU, USB, BIOS/UEFI, and can use KVM to implement the processor natively.
KVM is a linux driver, the kernel needs to be compiled with this module but this module is only useful if the hardware supports it. At hardware level it needs extensions like Intel VT-x or AMD-V. So this turns the Linux kernel into a type-1 hypervisor. Type-1 also called bare metal hypervisor, gives almost native level performance like Xen hypervisor, type-2 is mostly software level emulation and then 
there is type-3, that is more of a hybrid like KVM/QEMU, that can perform both of the above, QEMU is simply a type-2 but combining with KVM turns it closer to type-1.
KVM is the cpu acceleration backend, without it qemu just simply fallbacks to TCG based emulation. KVM is very much Linux specific, though it has been ported earlier to Illumos, which then moved to FreeBSD based bhyve hypervisor.
NetBSD uses NVMM (NetBSD Virtual Machine Monitor) its own native hypervisor. QEMU has native support for it, like for Linux host we pass -enable-kvm, we use -accel-nvmm.
In both cases we need a simple check at /dev. For linux /dev/kvm needs to exist if not load the module, for NetBSD /dev/nvmm needs to be present.

Basically hypervisor is a software that creates and manages vm's by controlling cpu/memory/device access on their behalf(guest os).Type 0 runs on a privileged layer, basically ring 0 in terms of Intel's hardware.
Rings are basically like modes, but modern hardware has multiple modes other than just the krnel and user mode.
So existence of kvm module on Linux makes the whole OS along with QEMU act like a type 1. Xen is a different case.
```txt
https://wiki.xenproject.org/wiki/Main_Page
```
With kVM/QEMU , Linux host acted as a VMM/hypervisor, it had the control of starting/stopping the hypervisor. In case on Xen, it calls the guest OS running on it as "domains". A special OS is tagged dom0, that can control the hypervisor and start/stop other guest OS's.
From the Xen wiki, the hypervisor supports two primary types of virtualization: paravirtualization (PV) and hardware virtualized machine (HVM) also known as “full virtualization”. Paravirtualization uses modified guest operating systems that we refer to as "enlightened" guests. These operating systems are aware that they are being virtualized and as such don’t require virtual hardware devices. Instead they make special calls to the hypervisor that allow them to access CPUs, storage and network resources.
In contrast, HVM guests need not be modified, as the hypervisor will create a fully virtual set of hardware devices for the machine resembling a physical x86 computer. This emulation requires more overhead than the paravirtualization approach but allows unmodified guest operating systems like Microsoft Windows to run on top of the hypervisor. HVMs are supported through virtualization extensions in the CPU. Several iterations of these extensions have been introduced in the last decade or so, collectively known as Intel VT and AMD-V and development continues. The technology is now prevalent; all recent servers, many desktops and some mobile systems should be equipped with at least some extensions.
Xen virtualization is now seen as on a spectrum, with PV at one end and HVM at the other. In between are various enhancements to improve performance: HVM with PV drivers, PVHVM or “Paravirtualization on HVM”, and most recently PVH.
Since KVM is bolted at the Linux kernel itself, Xen doesn't work like that, it has its own minimal microkernel like layer that boots before any OS,then it starts a privileged domain dom0, which starts other VM's.
To check if the existing kernel has Xen support:
```bash
zgrep XEN /proc/config.gz
```
In my case, output looks like this and the kernel supports it.
```bash
└[~]> zgrep XEN /proc/config.gz
CONFIG_XEN=y
CONFIG_XEN_PV=y
CONFIG_XEN_512GB=y
CONFIG_XEN_PV_SMP=y
CONFIG_XEN_PV_DOM0=y
CONFIG_XEN_PVHVM=y
CONFIG_XEN_PVHVM_SMP=y
CONFIG_XEN_PVHVM_GUEST=y
CONFIG_XEN_DEBUG_FS=y
CONFIG_XEN_PVH=y
CONFIG_XEN_DOM0=y
CONFIG_XEN_PV_MSR_SAFE=y
CONFIG_PCI_XEN=y
CONFIG_KVM_XEN=y
CONFIG_NET_9P_XEN=m
CONFIG_XEN_PCIDEV_FRONTEND=m
CONFIG_XEN_BLKDEV_FRONTEND=m
CONFIG_XEN_BLKDEV_BACKEND=m
CONFIG_XEN_SCSI_FRONTEND=m
CONFIG_NETXEN_NIC=m
CONFIG_XEN_NETDEV_FRONTEND=m
CONFIG_XEN_NETDEV_BACKEND=m
CONFIG_INPUT_XEN_KBDDEV_FRONTEND=m
CONFIG_HVC_XEN=y
CONFIG_HVC_XEN_FRONTEND=y
# CONFIG_TCG_XEN is not set
CONFIG_XEN_WDT=m
# CONFIG_DRM_XEN_FRONTEND is not set
CONFIG_XEN_FBDEV_FRONTEND=y
CONFIG_SND_XEN_FRONTEND=m
CONFIG_USB_XEN_HCD=m
CONFIG_MMC_SDHCI_XENON=m
CONFIG_XEN_BALLOON=y
CONFIG_XEN_BALLOON_MEMORY_HOTPLUG=y
CONFIG_XEN_MEMORY_HOTPLUG_LIMIT=512
CONFIG_XEN_SCRUB_PAGES_DEFAULT=y
CONFIG_XEN_DEV_EVTCHN=m
CONFIG_XEN_BACKEND=y
CONFIG_XENFS=m
CONFIG_XEN_COMPAT_XENFS=y
CONFIG_XEN_SYS_HYPERVISOR=y
CONFIG_XEN_XENBUS_FRONTEND=y
CONFIG_XEN_GNTDEV=m
CONFIG_XEN_GRANT_DEV_ALLOC=m
# CONFIG_XEN_GRANT_DMA_ALLOC is not set
CONFIG_SWIOTLB_XEN=y
CONFIG_XEN_PCI_STUB=y
CONFIG_XEN_PCIDEV_BACKEND=m
# CONFIG_XEN_PVCALLS_FRONTEND is not set
# CONFIG_XEN_PVCALLS_BACKEND is not set
CONFIG_XEN_SCSI_BACKEND=m
CONFIG_XEN_PRIVCMD=m
CONFIG_XEN_PRIVCMD_EVENTFD=y
CONFIG_XEN_ACPI_PROCESSOR=m
# CONFIG_XEN_MCE_LOG is not set
CONFIG_XEN_HAVE_PVMMU=y
CONFIG_XEN_EFI=y
CONFIG_XEN_AUTO_XLATE=y
CONFIG_XEN_ACPI=y
CONFIG_XEN_SYMS=y
CONFIG_XEN_HAVE_VPMU=y
CONFIG_XEN_FRONT_PGDIR_SHBUF=m
CONFIG_XEN_UNPOPULATED_ALLOC=y
CONFIG_XEN_GRANT_DMA_OPS=y
CONFIG_XEN_VIRTIO=y
# CONFIG_XEN_VIRTIO_FORCE_GRANT is not set


# clearly CONFIG_XEN_DOM0=y, CONFIG_XEN_PV=y, CONFIG_XEN_PVHVM=y is set, hence the kernel has builtin support.
```
Mechanisms that enable guest and host communication -:

1. MMIO/PIO - memory mapped I/O or port I/O, it is the old/boomer solution, the guest thinks its talking to real hardware registers, when it reads/writes those addresses,the CPU traps out of guest mode (via KVM), returns control to QEMU, then QEMU's device emulation code interprets what the guest wanted and updates emulated device state.
This is the mechanism behind the e1000 Network Interface Controller, ide disk controller works. Qemu pretends to be real hardware down to register level.

2. Virtio - This is the modern alternative, like virtio-net-pci, virtio-gpu-pci, virtio-rng. Instead of pretending to be real hardware, virtio is a paravirtualized interface, th guest OS clearly knows its talking to a virtualised environment and uses a purpose built protocol: shared memory ring buffers(virtqueues) that both guest and QEMU can read/write to.Also there exists a lightweight notification mechanism called ioeventfd/irqfd, kernel level avoids trapping to qemu userspace for notifcation itself.This
method has far less overhead than MMIO/PIO, which emulates hardware at register level.So only thing is NetBSD needs virtio guest drivers like viornd, vioif, etc.

Modern systems invlove heavy usage and presence of GPU's. We need is virgl/virglrenderer, its layered on top of virtio-gpu. The guest's Mesa driver will serialize the OpenGL/Gallium commands to a command buffer, write it to the virtqueue like other virtio traffic. Now the host will read the command buffer and display it as a real OpenGL/Vulkan calls against the host's GPU driver. In my case it is IRISxe.
virglrenderer, is a library that qemu links itself to is what reads the command buffer.
So, the communication channel is virtio(shared memory + virtqueue), and virglrenderer is the translator sitting on the host end.
Parameters like -display gtk,gl=on is important, as the display backend will need its own GL context to composite the result. gl=on, tells QEMU's GTK frontend to set that up.

The similar mechanism is followed for sound, virtio-sound is the interface, its newer compared to others and has lesser universal support, truth being QEMU still mostly relies on semi-emulated devices like intel-hda (emulating a real Intel HDA controller), paired with a host audio backend like -audiodev-pa for pulseaudio, others like alsa,sdl also exists. The guest HDA driver follows the MMIO path. Then QEMU translates it to calls against the host's audio system.
Another aspect is existence of a shared folder. It uses different protocol, the Plan 9's filesystem protocol, carried over virtio (virtio-9p) as the transport.
Guest mounts it as a real filesystem (mount_9p), the only difference here is every file operation (read/write/open/stat) is serialized as a 9p protocol message, gets sent to virtqueue, and qemu translates it into a real syscall against the host directory listed by the -virtfs parameter.

So the pattern we follow is if any virtio* variant exists, we use it or else we go to MMIO mechanism.
Other parameters that we might use, is virtio-blk-pci instead of default IDE style -drive interface. Useful for faster disk I/O for heavy build workloads. We need is vioblk driver support for the host, in my case NetBSD.
Next in the line is actual core counts and threads, we have to use -smp cores=4,threads=1 style topology control. We will need this while creating our own pkgsrc builds.
For even heavier usage we might need -object memeory-backend-memfd + hugepages.

The core part, is how exactly is the host going to emulate a hardware it doesn't even support. NetBSD doesn't have any support for raptor lake series and IRISxe integrated graphics card.
For the most part processors and GPU's are never written from scratch nowadays, they follow a template that reamins fairly common across generations, the only factors that change is introduction of newer parameters/options availability. Something else that might change is how the template is being handled at the backend.
Say for GPU's, modern Intel GPU's like IRISxe and ARC, use gallium drivers, it is a core part of MESA library's 3D project, fairly newer ones like IRISxe handles these gallium calls via OpenGL.
The gallium3D architecture was introduced in 2008 by Tungsten graphics to modernize MESA 3D project, so its like a protocol. Older Intel GPUs relied on i965 driver, so the community ported the older drivers to gallium via crocus project, because those drivers were hard to maintain.So from Intel's Gen8 (Broadwell) to current IRISxe these use iris drivers based on gallium3D project. Now vulkan is a different case,it uses the Intel's ANV driver. Gallium was developed to trace older API's of OpenGL, but vulkan is low level, where engines need to manage their own memeory and scheduling. Vulkan drivers bypass the Gallium drivers to talk to the hardware directly.

Similarly this remains true across processors. All these generations are based on X86_64 ISA. Thus instruction set across them have to be the same, so they execute the same byte codes. What differs is the extended instruction set support, like AVX2, AVX-512, SIMD extensions. So say a binary compiled with AVX-512 will crash on older generation (SIGILL) which doesn't have any such support. That differetiates compiler flags like -march=native and -march=x86-64-v2, the latter targetting feature level and former targeting a specific chip.
Another difference is microarchitecture and CPUID. The internal implementations like pipeline depth, cache sizes, branch predictor design, core/thread topology, these execute at different speeds, so performance and efficiency varies.
CPUID is the mechanism by which any software (OS, hypervisor and even userspace) queries what instructions a CPU has, something like a compiler or JIT calls CPUID to discover what is it running on, to choose the execution path.
Thats the sole reason a flag like -cpu host in qemu script is important.

So how is NetBSD able to boot as a VM but fails to boot from a live environment or I should rather say boot image, as NetBSD doesn't really have a traditional linux like live environment,though users can use console by exiting the installer.
The fact being its well known to me the NetBSD media, fails to recognize my NVME disk, so clearly on bleeding edge hardware CPU never fails but its the peripherals, NetBSD's tree just doesn't have any drivers for these peripherals, any these prripherals can be anything from NVME disk controller, to wifi chipset, USB4/thunderbolt controller, exact ACPI quirk table, exact PCIe root table,also secure boot interactions. NetBSD's media just is unable to parse ACPI on the table my device firmware emits, even possibility is install ,edia can't get display output through the specific GPU's boot time framebuffer path because of KMS/UEFI GOP mismatch.
Its just the kernel can't talk to the actual hardware its trying to boot on. So by virtualization, the hypervizor like KVM/QEMU or Xen never expose these peripherals, unless we detect some peripherals are supported i.e the drivers exist in the tree/kernel, so we decide to do a PCI passthrough. They try to pass a decade old stable virtual chipset like QEMU's q35/i440fx, emulated ICH9/PIIX4 AHCI disk controller (or virtio-blk), emulated e1000 NIC (or virtio-net), a standard VGA or virtio-gpu framebuffer for boot output.
Same goes with Xen, Xen HVM guests use qemu too, it just runs as qemu-dm inside dom0, providing the exact virtio* interfaces. Its a policy that Xen choose because maintaining such virtual peripherals is a huge ask, which QEMU already solved. While for Xen PV guests, which is Xen's older mode, came before KVM, Xen uses its own seperate paravirt device protocol that involves Xen PV drivers, that has a frontend and a backend. Frontend in the guest and backend in dom0.
Architecturally similar to virtio(shared memory ring buffers + lightweight notification), it was Xen's own independently designed protocol. QEMU's virtio was created later and partially the design idea was influenced from Xen's PV, to be hypervisor agnostic usable by KVM or Xen.
From my earlier output of: 
```bash
zgrep XEN /proc/config.gz

CONFIG_XEN_BLKDEV_FRONTEND/BACKEND
CONFIG_XEN_NETDEV_FRONTEND/BACKEND
CONFIG_XEN_PVH=y
```
Means we have those preconfigured.
Modern Xen deployments use PVH mode, which is enabled in our kernel config.

Most importantly if the hardware is supported, its better we utilise it directly to get native performance rather than using some vir* interface, the mechanism is PCI passthrough. Specifically in case of say CPU, we can use KVM/QEMU to run it natively via VT-x. The CPU has two modes VMX root(hypervisor mode) and VMX non-root(guest mode). When NetBSD executes a normal instruction like arithmetic or memory access, we can choose to run it directly on the physical Raptor Lake core, at native speed, that requires no translation via qemu's TCG.
The CPU runs in some elevated mode while running KVM/QEMU, its lower than kernel mode but miles higher than user mode, so if the guest tries something that requires privileged like writing to a control register or doing raw I/O port access, touching interrupt controllers, the CPU trips up a trap or interrupt(non-maskable) to call a VM-exit. This then hands control to the hypervisor(Xen/KVM) which decides what to do, then resumes the guest(VM-entry). So its basically balancing between running on raw hardware and emulating when boundaries cross.
The way to implement the PCI passthrough differs. With KVM/QEMU it uses VFIO (Virtual function I/O) passthrough, Xen uses the same mechanism, but the policy differs. It doesn't use Linux's VFIO framework, but has its own implementation.
Xen has PCI passthrough configured in guest's domain config file used by x1 create by the pci directive.
```bash
pci = [ '0000:01:00.0' ]
```
Format being (bus:device.function of target device, same as lspci shows).
For KVM/QEMU the IOMMU (Input output memory management unit) lets a specific PCIe device be isolated into its own protection domain and mapped to guest's address space. The guest talks to real device registers directly, with IOMMU enforcing that it can only DMA into memory which has been granted, so it protects the host from a misbehaving/malicious guest driver.
Requirement is IOMMU enabled in firmware, VT-d in UEFI.
On Intel we need to check:
```bash
[~]> sudo dmesg | grep -i iommu

[    0.077643] DMAR-IR: IOAPIC id 2 under DRHD base  0xfed91000 IOMMU 1
[    0.417408] pci 0000:00:02.0: DMAR: Skip IOMMU disabling for graphics
[    0.469648] iommu: Default domain type: Translated
[    0.469648] iommu: DMA domain TLB invalidation policy: lazy mode
[    0.588447] DMAR: Intel-IOMMU force enabled due to platform opt in
[    0.588687] pci 0000:00:02.0: Adding to iommu group 0
[    0.588716] pci 0000:00:00.0: Adding to iommu group 1
[    0.588723] pci 0000:00:04.0: Adding to iommu group 2
[    0.588734] pci 0000:00:06.0: Adding to iommu group 3
[    0.588743] pci 0000:00:07.0: Adding to iommu group 4
[    0.588750] pci 0000:00:08.0: Adding to iommu group 5
[    0.588760] pci 0000:00:0d.0: Adding to iommu group 6
[    0.588766] pci 0000:00:0d.2: Adding to iommu group 6
[    0.588774] pci 0000:00:0e.0: Adding to iommu group 7
[    0.588781] pci 0000:00:12.0: Adding to iommu group 8
[    0.588792] pci 0000:00:14.0: Adding to iommu group 9
[    0.588798] pci 0000:00:14.2: Adding to iommu group 9
[    0.588807] pci 0000:00:14.3: Adding to iommu group 10
[    0.588817] pci 0000:00:15.0: Adding to iommu group 11
[    0.588824] pci 0000:00:15.1: Adding to iommu group 11
[    0.588832] pci 0000:00:16.0: Adding to iommu group 12
[    0.588845] pci 0000:00:1f.0: Adding to iommu group 13
[    0.588852] pci 0000:00:1f.3: Adding to iommu group 13
[    0.588859] pci 0000:00:1f.4: Adding to iommu group 13
[    0.588866] pci 0000:00:1f.5: Adding to iommu group 13
[    2.176329] pci 10000:e0:06.0: Adding to iommu group 7
[    2.177396] pci 10000:e1:00.0: Adding to iommu group 7

[~]> sudo lspci -nnk
0000:00:00.0 Host bridge [0600]: Intel Corporation Raptor Lake-P/U 4p+8e cores Host Bridge/DRAM Controller [8086:a707]
        Subsystem: Dell Device [1028:0c14]
        Kernel driver in use: igen6_edac
        Kernel modules: igen6_edac
0000:00:02.0 VGA compatible controller [0300]: Intel Corporation Raptor Lake-P [Iris Xe Graphics] [8086:a7a0] (rev 04)
        Subsystem: Dell Device [1028:0c14]
        Kernel driver in use: i915
        Kernel modules: i915, xe
0000:00:04.0 Signal processing controller [1180]: Intel Corporation Raptor Lake Dynamic Platform and Thermal Framework Processor Participant [8086:a71d]
        Subsystem: Dell Device [1028:0c14]
        Kernel driver in use: proc_thermal_pci
        Kernel modules: processor_thermal_device_pci
0000:00:06.0 System peripheral [0880]: Intel Corporation RST VMD Managed Controller [8086:09ab]
0000:00:07.0 PCI bridge [0604]: Intel Corporation Raptor Lake-P Thunderbolt 4 PCI Express Root Port #0 [8086:a76e]
        Subsystem: Dell Device [1028:0c14]
        Kernel driver in use: pcieport
        Kernel modules: shpchp
0000:00:08.0 System peripheral [0880]: Intel Corporation GNA Scoring Accelerator module [8086:a74f]
        Subsystem: Dell Device [1028:0c14]
0000:00:0d.0 USB controller [0c03]: Intel Corporation Raptor Lake-P Thunderbolt 4 USB Controller [8086:a71e]
        Subsystem: Dell Device [1028:0c14]
        Kernel driver in use: xhci_hcd
        Kernel modules: xhci_pci
0000:00:0d.2 USB controller [0c03]: Intel Corporation Raptor Lake-P Thunderbolt 4 NHI #0 [8086:a73e]
        Subsystem: Dell Device [1028:0c14]
        Kernel driver in use: thunderbolt
        Kernel modules: thunderbolt
0000:00:0e.0 RAID bus controller [0104]: Intel Corporation RST Volume Management Device Controller [8086:a77f]
        Subsystem: Dell Device [1028:0c14]
        Kernel driver in use: vmd
        Kernel modules: vmd
0000:00:12.0 Serial controller [0700]: Intel Corporation Alder Lake-P Integrated Sensor Hub [8086:51fc] (rev 01)
        Subsystem: Dell Device [1028:0c14]
        Kernel driver in use: intel_ish_ipc
        Kernel modules: intel_ish_ipc
0000:00:14.0 USB controller [0c03]: Intel Corporation Alder Lake PCH USB 3.2 xHCI Host Controller [8086:51ed] (rev 01)
        Subsystem: Dell Device [1028:0c14]
        Kernel driver in use: xhci_hcd
        Kernel modules: xhci_pci
0000:00:14.2 RAM memory [0500]: Intel Corporation Alder Lake PCH Shared SRAM [8086:51ef] (rev 01)
        Subsystem: Dell Device [1028:0c14]
0000:00:14.3 Network controller [0280]: Intel Corporation Raptor Lake PCH CNVi WiFi [8086:51f1] (rev 01)
        Subsystem: Intel Corporation Wi-Fi 6E AX211 160MHz [8086:4090]
        Kernel driver in use: iwlwifi
        Kernel modules: iwlwifi
0000:00:15.0 Serial bus controller [0c80]: Intel Corporation Alder Lake PCH Serial IO I2C Controller #0 [8086:51e8] (rev 01)
        Subsystem: Dell Device [1028:0c14]
        Kernel driver in use: intel-lpss
        Kernel modules: intel_lpss_pci
0000:00:15.1 Serial bus controller [0c80]: Intel Corporation Alder Lake PCH Serial IO I2C Controller #1 [8086:51e9] (rev 01)
        Subsystem: Dell Device [1028:0c14]
        Kernel driver in use: intel-lpss
        Kernel modules: intel_lpss_pci
0000:00:16.0 Communication controller [0780]: Intel Corporation Alder Lake PCH HECI Controller [8086:51e0] (rev 01)
        Subsystem: Dell Device [1028:0c14]
        Kernel driver in use: mei_me
        Kernel modules: mei_me
0000:00:1f.0 ISA bridge [0601]: Intel Corporation Raptor Lake LPC/eSPI Controller [8086:519d] (rev 01)
        Subsystem: Dell Device [1028:0c14]
0000:00:1f.3 Multimedia audio controller [0401]: Intel Corporation Raptor Lake-P/U/H cAVS [8086:51ca] (rev 01)
        Subsystem: Dell Device [1028:0c14]
        Kernel driver in use: sof-audio-pci-intel-tgl
        Kernel modules: snd_soc_avs, snd_sof_pci_intel_tgl, snd_hda_intel
0000:00:1f.4 SMBus [0c05]: Intel Corporation Alder Lake PCH-P SMBus Host Controller [8086:51a3] (rev 01)
        Subsystem: Dell Device [1028:0c14]
        Kernel driver in use: i801_smbus
        Kernel modules: i2c_i801
0000:00:1f.5 Serial bus controller [0c80]: Intel Corporation Alder Lake-P PCH SPI Controller [8086:51a4] (rev 01)
        Subsystem: Dell Device [1028:0c14]
        Kernel driver in use: intel-spi
        Kernel modules: spi_intel_pci
10000:e0:06.0 PCI bridge [0604]: Intel Corporation Raptor Lake PCI Express 4.0 Graphics Port [8086:a74d]
        Subsystem: Dell Device [1028:0c14]
        Kernel driver in use: pcieport
        Kernel modules: shpchp
10000:e1:00.0 Non-Volatile memory controller [0108]: Sandisk Corp PC SN740 NVMe SSD (DRAM-less) [15b7:5015] (rev 01)
        Subsystem: Sandisk Corp PC SN740 NVMe SSD (DRAM-less) [15b7:5015]
        Kernel driver in use: nvme
        Kernel modules: nvme

```
So we have to find the vendor:devie ID, then bind it to vfio-pci, either using kernel driver ovverride at boot or driverctl/manual echo to /sys/bus/pci/drivers/vfio-pci/bind

Also IOMMU device groups matter, devices in the same group gets passed through together, there is no option to split them.
An example being 
```bash
└[~]> sudo find /sys/kernel/iommu_groups/ -maxdepth 1 -type d
/sys/kernel/iommu_groups/
/sys/kernel/iommu_groups/7
/sys/kernel/iommu_groups/5
/sys/kernel/iommu_groups/13
/sys/kernel/iommu_groups/3
/sys/kernel/iommu_groups/11
/sys/kernel/iommu_groups/1
/sys/kernel/iommu_groups/8
/sys/kernel/iommu_groups/6
/sys/kernel/iommu_groups/4
/sys/kernel/iommu_groups/12
/sys/kernel/iommu_groups/2
/sys/kernel/iommu_groups/10
/sys/kernel/iommu_groups/0
/sys/kernel/iommu_groups/9
┌[honken@Pratyush-PC] [/dev/pts/4]
└[~]> sudo ls /sys/kernel/iommu_groups/12/devices
0000:00:16.0
┌[honken@Pratyush-PC] [/dev/pts/4]
└[~]> sudo ls /sys/kernel/iommu_groups/7/devices
0000:00:0e.0  10000:e0:06.0  10000:e1:00.0
```
The only qemu flag in need is -device vfio-pci,host=<bus:device.function>
In my case I can clearly see my IRISxe graphics card is in iommu_group0 and I'm not going to do a PCI passthrough for it, because I may loose my own display. Its a well known cavaet of using integrated graphics card, they share the VRAM with device RAM, PCI passthrough will handover the peripheral to the guest.
Same isn't true for CPU cores, they don't belong to iommu_groups. So no DMA and PCIe bus for them.
If we want more control or predictible behaviour over CPU then vCPU pinning is required. It makes use of taskset or QEMU's own -numa/CPU affinity options, which restricts which physical cores each vCPU can run on. Its solely for controlling performance.
So a peripheral that being used in PCI passthrough is inaccessible to the host for that time in case of KVM/QEMU and in case of Xen inaccessible to dom0.

For writing the installer script, we need two bash scripts, one boots the NetBSD image into the QEMU disk, and second boots into the disk.
I call the 1st one boot_install.sh and 2nd one boot_normal.sh

```bash
# create a qemu disk image to install the NetBSD image, I will allocate myself 100G
mkdir -p ~/virtual-machines/NetBSD
cd ~/virtual-machines/NetBSD
qemu-img create -f qcow2 netbsd.img 100G
# qcow2 is qemu copy on write version 2

cat ./boot_install.sh
#! /usr/bin/bash

qemu-system-x86_64 \
  -m 4G \
  -smp 4 \
  -cdrom  NetBSD-11.0-amd64.iso  \
  -drive file=netbsd.img,format=qcow2 \
  -boot d \
  -enable-kvm \
  -virtfs local,path=/home/honken/virtual-machines/NetBSD,mount_tag=hostshare,security_model=none \
  -netdev user,id=network-vm \
  -device virtio-net-pci,netdev=network-vm \
  -bios /usr/share/edk2/OvmfX64/OVMF_CODE.fd \
  -nographic


  cat ./boot_normal.sh
  #! /usr/bin/bash

qemu-system-x86_64 \
  -m 4G \
  -smp 4 \
  -drive file=netbsd.img,format=qcow2 \
  -boot c \
  -enable-kvm \
  -virtfs local,path=/home/honken/virtual-machines/NetBSD,mount_tag=hostshare,security_model=none \
  -netdev user,id=network-vm,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=network-vm \
  -bios /usr/share/edk2/OvmfX64/OVMF_CODE.fd \
  -object rng-random,filename=/dev/urandom,id=viornd0 \
  -device virtio-gpu-gl-pci \
  -nographic
  ```
  Now we make them executable 
  ```bash
  chmod +x boot_install.sh
  chmod +x boot_normal.sh
  ```

  Launch the installer, configure and then boot up the VM
  ```
  ./boot_install.sh
  ./boot_normal.sh
```
