---
layout: post
title: "Starting with NetBSD and fixing QEMU flags"
date: 2026-08-17
categories: System Architecture
---

In the last post I jotted down my understanding of Virtualization and how hypervisors like Xen and KVM/QEMU works, along with working scripts.
Considering the fact that my initial focus is to understand the whole NetBSD layout,its init system and rc scripts, how its package manager pkgin functions, the ports system of pkgsrc, though I shouldn't be using the term ports for NetBSD.
Further components like pkg_add, pkg_delete, pkg_info along with how the pkgsrc tree tries to be OS agnostic, the NetBSD's make known to everyone as "bmake".
Working with NetBSD's sndio drivers and delving into kernel code. Then I would delve into rump kernels. With my main focus on resolving the fact that despite NetBSD source tree having a stable NVME driver, the installer media doesn't recognize it.
Same issue persists with OpenBSD, with only FreeBSD working without any caveats. So understanding the FreeBSD specific driver and then I will try to port it to NetBSD.
But foremost I need to fix the issue with entropy and shared directory with KVM/QEMU. From now on the main refernece guide will be NetBSD manual pages.

So earlier I had done a custom installation of NetBSD selecting different sets and installing them from installation media and importing rest from NetBSD's CDN via http.
The sets for reference were kernel(generic), discarding kernel(generic_kaslr) which is the same generic kernel with kernel address space randomization enabled, loadable kernel modules(.kmod files), base(mandatory and required ofc), base 32-bit compatibility(I disabled this because I'm not going to working with 32 bit binaries and legacy software but good to have for daily drivers),config files(/etc), compiler tools(Needed for developemt), 
Games(old BSD TUI games, not required ), Man pages(troff formatted, gold source of info), Man pages HTML (pre-rendered as HTML instead of troff ), graphics driver firmware(I disabled this, because initially my work is solely TUI based and getting NVME and other peripherals work and getting non-llvmpipe display is my last concern for now), misc(no idea honestly, probably some extra locales, good to have ), recovery tools(Enabled, better than runnnig the installation media again), test programs(enabled, as mostly required for my VMD work), text processing tools(must for troff based man pages), x11 sets(disabled), source and debug sets(installs /usr/src, anyways needed for devel work, better to have now than installing later from github mirror, for debugging we get tools like gdb for a panic kernel).
Further I enabled sshd(because using with -nographic means dealing with vt100 or vt220 terminal, they are fine for physical hardware but the buffering isn't great when virtualising, even setting stty rows and cols, rendering is unpredictable. It sometimes fills screen, sometimes display only in half, so ssh is useful for visual satisfaction pov), enabling ntpd, The ntpd utility is an operating system daemon which sets and maintains the system time of day in synchronism with Internet standard time servers.
Then ntpdate,  ntpdate sets the local date and time by polling the Network Time Protocol (NTP) server(s) given as the server arguments to determine the correct time. It must be run as root on the local host, better to leave it disabled, leans more towards legacy outdated tech and not much relevant to kernel work.
Other options like multicast dns and cgd(similar to Linux's LUKS encryption), LVM and raidframe disabled because I don't require redundancy and disk flexibility on qemu virtual disks.
Also the reasoning behind, the irregular display with -nographic is ,there is no monitor hence no pixel resolution, no window for qemu to manage, qemu's serial output just gets piped to whatever terminal application I launched with boot_normal.sh, in my case ghostty.
So the screen filling half/full is a rendering issue of the terminal emulator's own window behaving inconsistently, window manager auto sizing, terminal app remembering a stale size, etc. Its nothing on qemu for fixing it.
So a real serial console has no way to report the guest OS, what size is my terminal emulator. So setting stty sizes will also not help, but in case of ssh, it negotiates window size(TIOCWINSZ) and sends SIGWINCH automatically on resize, thus with -nographic the serial console just uses the default which most likely is 80x24.

Also while booting via serial console with -nographic simply choosing normal boot via 1, will boot the system but lack serial port because it expects a VGA display there.
So boot using option 3, and use the boot parameters
```bash
consdev com0
boot
```
Upstream source for all QEMU flags are listed here:
```txt
https://wiki.qemu.org/Features/
```
Now comes the entropy part, as soon as I boot into the system I'm greeted with 
```bash
Welcome to NetBSD!
-- UNSAFE KEYS WARNING:
        The ssh host keys on this machine have been generated with
        not enough entropy configured, so they may be predictable.
        To fix, follow the "Adding entropy" section in the entropy(7)
        man page.  After this machine has enough entropy, re-generate
        the ssh host keys by running:
                /etc/rc.d/sshd keyregen
```
Honestly in the installation phase itself NetBSD will complain lack of entropy and advise either to generate it or fetch it via network, for a VM its fine but still a secure practise to follow on a development environment.
Entropy is random unpredictable secrets needed for security, 
```bash
man entropy     # hardware random number generators based on thermal noise in silicon circuits
                # others may require operator intervention for security.
```
```txt
Features/VirtIORNG, QEMU Wiki, https://wiki.qemu.org/Features/VirtIORNG  # from NetBSD man page
```
In our case we don't do a passthrough, that leaves the host vulnerable for the time being the guest is active, so we simply use -device virtio-rng-pci.
From qemu's wiki -device virtio-rng-pci to the QEMU invocation will add the device with a default host backend. As of QEMU 1.3, the default backend is to use the host's /dev/random as a source of entropy.
To modify this source to a real hardware RNG on the host, use:
```txt
-object rng-random,filename=/dev/hwrng,id=rng0 \
-device virtio-rng-pci,rng=rng0

# optional parameter to limit the rate of data sent to the guest
-device virtio-rng-pci,max-bytes=1024,period=1000
```
As its a guest mostly running on userspace, I prefer passing /dev/urandom instead of /dev/hwrng.
/dev/urandom is a fast software-based cryptographic pseudorandom generator meant for general applications, while /dev/hwrng provides raw, direct access to a physical hardware random number generator
At the end /dev/hwrng writes to /dev/random, if you trust hardware better use /dev/hwrng.
Modifying the boot_normal.sh, requires a device shutdown and poweron for the guest, a mere reboot doesn't seem to work.
In NetBSD to shutdown use
```bash
halt -p    # As root or use doas
```
To see if changes are taking place inside the NetBSD guest, 
```bash
netbsd_devel$ doas rndctl -l
Source       Estimated bits    Samples Type   Flags
/dev/random               0          0 ???    collect, v
cd0                       0          0 disk   collect, v, t
wd0                       0       6701 disk   collect, v, t
fd0                       0          0 disk   collect, v, t
hardclock                 0      19029 skew   collect, t
pms0                      0          0 tty    collect, v, t
pckbd0                    0          0 tty    collect, v, t
system-power              0          0 power  collect, v, t
autoconf                  0        152 ???    collect, t
seed                      0          1 ???    collect, v
uvmfault                  0        502 vm     collect, v, t

# no viornd0 listed, so changes haven't taken place requires halt.
```
Additionally we may check what flags QEMU launched with using
```bash
[~]> pgrep -fa qemu-system-x86_64
1227820 qemu-system-x86_64 -m 4G -smp 4 -drive file=netbsd.img,format=qcow2 -boot c -enable-kvm -virtfs local,path=/home/honken/virtual-machines/NetBSD,mount_tag=hostshare,security_model=none -netdev user,id=network-vm,hostfwd=tcp::2222-:22 -device virtio-net-pci,netdev=network-vm -bios /usr/share/edk2/OvmfX64/OVMF_CODE.fd -object rng-random,filename=/dev/urandom,id=viornd0 -nographic
```
To confirm the device is visible at the PCI level inside NetBSD we use pcictl and dmesg remains a traditional source.
```bash
netbsd_devel$ pcictl pci0 list
000:00:0: Intel 82441FX (PMC) PCI and Memory Controller (host bridge, revision 0x02)
000:01:0: Intel 82371SB (PIIX3) PCI-ISA Bridge (ISA bridge)
000:01:1: Intel 82371SB (PIIX3) IDE Interface (IDE mass storage, interface 0x80)
000:01:3: Intel 82371AB (PIIX4) Power Management Controller (miscellaneous bridge, revision 0x03)
000:02:0: vendor 1234 product 1111 (VGA display, revision 0x02)
000:03:0: Qumranet Virtio 9p Filesystem (prehistoric, subclass 0x02)
000:04:0: Qumranet Virtio Network (ethernet network)
000:05:0: Qumranet Virtio RNG Entropy (prehistoric, subclass 0xff)
netbsd_devel$ doas rndctl -l
Source       Estimated bits    Samples Type   Flags
/dev/random               0          0 ???    collect, v
cd0                       0          0 disk   collect, v, t
wd0                       0       6318 disk   collect, v, t
fd0                       0          0 disk   collect, v, t
hardclock                 0        499 skew   collect, t
viornd0                 512          2 rng    estimate, collect, v
pms0                      0          0 tty    collect, v, t
pckbd0                    0          0 tty    collect, v, t
system-power              0          0 power  collect, v, t
autoconf                  0        156 ???    collect, t
seed                      0          1 ???    collect, v
uvmfault                  0        434 vm     collect, v, t            

# clearly now viornd0 is present after a fresh boot.
```
More info using dmesg, if still not present we can check for viornd driver support in kernel, if not fetch using pkgin and use modload to load a module.
```bash
dmesg | grep -i -E "virtio|viornd|rnd"
modstat | grep -i virtio
modload viornd
```
Then after entropy is generated, we regenerate ssh keys and clear stale host keys, then connect via ssh.
```bash
/etc/rc.d/sshd keyregen
ssh-keygen -R "[localhost]:2222"       # this is the port I opened for the guest, passed via qemu flag
ssh -p 2222 pratyush@localhost         # guest user@localhost
```


Since I can't always rely on networking to fetch files via some remote repository, I need hostshare for quick access to large files from host to guest.
So I attach the VM directory for hostshare, and this acts as an attached disk that can be accessed from /mnt.
Earlier in my boot_normal.sh, I had used the QEMU flag
```txt
-virtfs local,path=/home/honken/virtual-machines/NetBSD,mount_tag=hostshare,security_model=none \
```
But the above flag has simply no effect, /mnt has nothing attached to it, because that flag isn't configured properly, it needs the protocol to be specified. Implemeting of a hostshare via virtfs is a mechanism but OS have different policy to reach that result.

```bash
[~]> pgrep -fa qemu-system-x86_64 | grep -o "mount_tag=[a-zA-Z0-9]*"
mount_tag=hostshare                                                                  
(pratyush)::[~] >> mount_9p hostshare /mnt
mount_9p: No address associated with hostname
(pratyush)::[~] >> doas mount_9p hostshare /mnt
mount_9p: No address associated with hostname
(pratyush)::[~] >> modstat | grep -i puffs
puffs                      vfs      builtin  -        0       - putter
(pratyush)::[~] >> modstat | grep -i 9p
vio9p                      driver   builtin  -        0       - virtio
(pratyush)::[~] >> doas modload puffs
modload: puffs: File exists
(pratyush)::[~] >> ls /mnt
```
We can check if the device is attached to PCI/drier level
```bash
pcictl pci0 list | grep -i 9p
dmesg | grep -i vio9p
```
From qemu docs:
```txt
https://www.qemu.org/docs/master/system/qemu-manpage.html
```
Virtfs defines a new virtual filesystem device and expose it to the guest using a virtio-9p-device (a.k.a. 9pfs), 
which essentially means that a certain directory on host is made directly accessible by guest as a pass-through file system by using the 9P network protocol for communication between host and guests, if desired even accessible, shared by several guests simultaneously.
virtfs is actually just a convenience shortcut for its generalized form -fsdev -device virtio-9p-pci.
"security_model=security_model" specifies the security model to be used for this export path. Supported security models are “passthrough”, “mapped-xattr”, “mapped-file” and “none”. In “passthrough” security model, files are stored using the same credentials as they are created on the guest. This requires QEMU to run as root. 
In “mapped-xattr” security model, some of the file attributes like uid, gid, mode bits and link target are stored as file attributes. For “mapped-file” these attributes are stored in the hidden .virtfs_metadata directory. Directories exported by this security model cannot interact with other unix tools. 
“none” security model is same as passthrough except the sever won’t report failures if it fails to set file attributes like ownership. Security model is mandatory only for local fsdriver.
Also by default the hostshare is read-write, but can be set to readonly=on.

More info on the 9plan file sharing protocol:
```txt
man mount_9p
man vio9p
https://ericvh.github.io/9p-rfc/
https://doc.cat-v.org/plan_9/4th_edition/papers/names   # use of namespaces, plan9s central idea uniform namespaces, everything is a file by Rob Pike et al
https://docs.kernel.org/filesystems/9p.html
https://github.com/chaos/diod/blob/master/protocol.md   # The 9P2000.L protocol spec itself, maintained in the v9fs project
https://github.com/NetBSD/src/tree/trunk/sys/fs/puffs   # client code under /sys/dev/pci
https://www.qemu.org/docs/master/system/qemu-manpage.html  # central doc, search for virtfs or fsdev

so apply changes to qemu flags in boot_normal.sh
```bash
-fsdev local,security_model=none,id=fsdev0,path=/home/honken/virtual-machines/NetBSD \
-device virtio-9p-pci,fsdev=fsdev0,mount_tag=hostshare
```
Then in the NetBSD guest
```bash
doas mount_9p -c -u /dev/vio9p0 /mnt
```
```bash
(pratyush)::[~] >> ls /mnt
NetBSD-10.1-amd64.iso                boot_install.sh                      netbsd-internals-en.pdf.gz           pkgsrc.pdf
NetBSD-11.0-amd64.iso                boot_normal.sh                       netbsd.img
NetBSD-11.0_RC2-amd64-dvd.iso        netbsd-en.pdf                        notes.txt
```
If dmesg shows vio9p0 is connected, but the device is still not present, then we must check if device node exists. If its missing then we create it using MAKEDEV, because some virtio device nodes aren't created and requires manual intervention via MAKEDEV.
```bash
ls -la /dev/vio9p*
cd /dev
doas ./MAKEDEV vio9p
```
The above process is highly unlikely, the virtfs mounting works only by plan9 file transfer protocol because of uniform namespaces.
This makes the VM ready for the NetBSD specific study.

As I had pkgin enabled while installing by installer, I will be installing git bash nvim, one crucial detail is pkgin fetches precompiled tarballs for the specific architecture from ftp://ftp.netbsd.org/pub/pkgsrc/packages/NetBSD/$arch/$branch/All.
These tarballs are built for a snapshot of pkgsrc, so a lot of time pkgin throws error because of dependency mismatch. In that case pkgsrc is the only option, and packages need to be compiled.
so for example installing links
```bash
cd /usr/pkgsrc/www/links
doas make install

# In case to search for a particular application just use
(pratyush)::[/usr/pkgsrc] >> find ./*/wgetpaste
./net/wgetpaste
./net/wgetpaste/CVS
./net/wgetpaste/CVS/Root
./net/wgetpaste/CVS/Repository
./net/wgetpaste/CVS/Entries
./net/wgetpaste/CVS/Tag
./net/wgetpaste/Makefile
./net/wgetpaste/DESCR
./net/wgetpaste/distinfo
./net/wgetpaste/PLIST
./net/wgetpaste/files
./net/wgetpaste/files/CVS
./net/wgetpaste/files/CVS/Root
./net/wgetpaste/files/CVS/Repository
./net/wgetpaste/files/CVS/Entries
./net/wgetpaste/files/CVS/Tag
./net/wgetpaste/files/wgetpaste.conf
./net/wgetpaste/patches
./net/wgetpaste/patches/CVS
./net/wgetpaste/patches/CVS/Root
./net/wgetpaste/patches/CVS/Repository
./net/wgetpaste/patches/CVS/Entries
./net/wgetpaste/patches/CVS/Tag
./net/wgetpaste/patches/patch-wgetpaste
(pratyush)::[/usr/pkgsrc] >> cd net/wgetpaste/
(pratyush)::[/usr/pkgsrc/net/wgetpaste] >> ls
CVS      DESCR    Makefile PLIST    distinfo files    patches
```
pkgsrc is NetBSD package collection,it is a framework for building and maintaining 3rd party software on NetBSD and other UNIX like systems, pkgsrc also works on DragonflyBSD, used to work on MINIX, SmartOS (an Illumos distribution). These have like 1st class support(informally based on use cases, though no such tiering from pkgsrc its highly portable).
The notable difference to linux is XDG naming, there are no ~/.config files created by default, these are freedesktop specific conventions, applications don't require it mandatorily, if it exists it reads it.
Same goes with /etc, there is a clean seperation of base system and user installed packages, base packages configs reside in /etc, whereas for 3rd party applications in /usr/pkg/etc, thus the base system remains clean across major releases.
As pkgsrc primarily uses bmake to compile and then pkg_add to install and pkg_delete to remove generated binaries, there needs a centralised config file to control compile time options, like gentoo's /etc/portage/make.conf.
In case of NetBSD its /etc/mk.conf. It follows the same pattern, a file doesn't exist unless its defaults needs to be overridden.
All the compile time options are listed in the mk.conf manpage, and the real defaults reside in /usr/pkgsrc/mk/defaults/mk.conf
Since this blog has already streched long, I would end it here, with next blog expalaining creation of a pkg and installation, then optimisations using LLVM/Clang, and methods to override the default gcc compiler.



