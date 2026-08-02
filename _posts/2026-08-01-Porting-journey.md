---
layout: post
title: "Porting android device to Postmarket OS (Part - I)"
date: 2026-08-1
categories: Porting documentation
---

Lately, I've been trying to port my android tablet to Postmarket OS, well PostmarketOS (from here on I will refer it as pmos) is a Linux distribution intended for mobile phones.
Well pmos lets a user run mainstream linux kernel along with the utilities used by common Linux distributions ranging from init system like systemd to complete desktop environments.
Though these utilities are tweaked heavily to suit mobile phone layouts, like Phosh (maintained by pmos community) basically works like gnome for phones, there exists others like plasma-mobile and many window managers.
This blog is meant for documenting my journey of my current ongoing work on porting my samsung galaxy tab A7, codenamed : gta4l to postmarket OS.

First, i would define some terminologies that basically defines the whole underlying technology that makes android different from mainstream Linux based distributions.
Starting from bootloader and partitions, most of the desktop os, lets you define partitions that includes a UEFI/BIOS partition, / (root partition), and an optional swap partition. This is usually a common use pattern, though a user can split the partitions based on utility, like creating a separate tmpfs, if a machine has
low memory and is suited for a compilation heavy environment. A separate /var if intended for a mail server. But android intended for handheld devices works in a heavy environment of embedded systems.
Thus it includes multiple partitions that holds binary blobs for hardware like camera, flash light, speakers and multiple such embedded hardware. 
Another example being bluetooth, Android uses its own bluetooth stack (Fluoride/Gabeldorsche), talking to hardware via a vendor HAL. Whereas pmos uses bluez. As linux distributions expect every code to exist in either the kernel or as loadable modules, this behaviour creates conflicts, and so lot of times peripherals just don't work in pmos. While BlueZ,which talks to hardware via HCI (Host Controller Interface),the standard protocol. The chip itself speaks HCI regardless of OS.
Therefore when a utility/software requires a camera access, the android kernel consists of the device driver, that has the ability to interact with the binary blob (firmaware) situated in the partition, these drivers then pass on the control to the binary, which performs the action.
Other than that android has another defining concept of slots, older devices and even many lower end devices make use of single slot mechanism (A slot), while latest and higher end devices make use of dual slots (A/B) to manage upgrades. 

While in general the android environment seems more inaccessible to changes compared to desktop counterparts, its actually the opposite. Android devices lets a super user have complete access including the bootloader, which if undergoes modification or accidental deletion makes the system unbootable, commonly referred as "hard bricking".
While soft bricking is recoverable, which includes flashing the wrong boot image. But scenarios exist where hard bricking is possible if a user flashes a wrong recovery and boot images. MOst important defining concept in android ecosystem being three images, vbmeta, dtbo, recovery.img and boot.img.
vbmeta checks for signature of boot.img, so if a wrong/unrealted device boot.img is flashed, android will refuse to boot and will take to recovery partition.
Next comes dtbo, which is a binary file, produced from dts (device tree source), it defines how the entire board is defined on the SoC(System on Chip).
As android being part of embedded system, it is meant to be space efficent. This dts tells the kernel, how the board is defined, which peripheral is connected to which port.
As Linux kernel is highly portable, it runs on many embedded hardware, so the concept of dts is not alien to the OS itself, the issue lies in dts naming conventions. 

Here, I will define two kernel sources, upstream and downstream. Upstream being the latest branch of linux kernel or a kernel compatible with kerenl.org's releases. Whereas downstream being the AOSP based kernel shipped by the vendor.
So the dts file structure defining is what decides portability. Understanding of the device board thats slapped on top of the SoC is crucial.

So to get the images, kernel source tree and dts files(though most of the vendors rarely release this, vendors mostly try to release as less as possible without violating the GPL licenced parts of Android).
Well we can always extract these images from a running system using adb(android debug bridge). Most vendors have a dedicated channel for releasing these software components, here we are trying to port a samsung galaxy tab A7, model : SM-T505, device codename : gta4l.
For samsung the dedicated release channel is :
```txt
https://opensource.samsung.com/
```
As I'm running Lineage OS currently on this tablet, its comparatively easier, as I can get all the sources from Lineage's git repo. This also has a massive advantage, that everytime I need not rely on odin/heimdall instead I can use fastbootd.
```txt
https://wiki.lineageos.org/devices/gta4l/
https://github.com/LineageOS/lineage_wiki/blob/main/_data/devices/gta4l.yml
```
The gta4l.yml file contains device specification and Lineage maintainer details, as Lineage provides complete working custom ROM for official devices, these config are highly reliable source.
```yml
architecture: arm64
battery: {capacity: 7040, removable: false, tech: 'Li-Po'}
before_install: {instructions: 'needs_specific_android_fw', lineage_version: 20, version: '12'}
before_recovery_install: samsung_qcom
bluetooth: {profiles: [A2DP], spec: '5.0'}
cameras:
  - {flash: None, info: '8 MP (Primary)'}
  - {flash: None, info: '5 MP (Front Facing)'}
codename: gta4l
cpu: Kryo 260
cpu_cores: '8'
cpu_freq: 4 x 2.0 GHz + 4 x 1.8 GHz
current_branch: 23.2
dimensions: {depth: 7, height: 157.4, width: 247.6}
download_boot: With the device powered off, hold <kbd>Volume Down</kbd> + <kbd>Volume Up</kbd> then connect USB cable to PC.
firmware_update: firmware_update_samsung_gta4l
gpu: Qualcomm Adreno 610
image: gta4l.png
install_method: samloader_rs
kernel: {repo: android_kernel_samsung_sm6115, version: '4.19'}
maintainers: [chrmhoffmann]
models: [SM-T505, SM-T505C, SM-T505N, SM-T507]
name: 'Galaxy Tab A7 10.4 2020 (LTE)'
network: [2G GSM, 2G CDMA, 3G UMTS, 4G LTE]
peripherals: [3.5mm jack, Accelerometer, Compass, GPS, Gyroscope, Light sensor, Proximity sensor, USB OTG]
quirks: [ims]
ram: 3 GB
recovery_boot: Reboot and immediately hold <kbd>Volume Up</kbd> + <kbd>Power</kbd> while the device is connected to a PC via USB cable.
recovery_partition_name: recovery
release: 2020-09
screen: {resolution: '2000x1200', size: 10.4, technology: 'TFT LCD'}
sdcard: {size_max: '1 TB'}
soc: Qualcomm SM6115 Snapdragon 662
storage: 32 GB
tree: android_device_samsung_gta4l
type: tablet
vendor: Samsung
vendor_short: samsung
versions: [20, 21, 22.1, 22.2, 23.0, 23.2]
wifi: 802.11 a/b/g/n/ac
```

For now the priority is to get a bootable image with display and touch sensors working, once we get to that stage, next focus would be on audio, bluetooth and getting a complete desktop environment working.
From the yml file, the SoC is "Qualcomm SM6115 Snapdragon 662".Thats clearly a good thing, because a SoC based on Qualcomm is most likely to have the device tree in linux mainline kernel, unlike mediatek which seems to be bit opaque or rather say uninterested.
So here I will list all the links that I will commonly refer to create and understand valid dts files and as well as look into its sibling devices that may be supported.
```txt
https://wiki.postmarketos.org/wiki/Samsung_Galaxy_Tab_A7_(samsung-gta4lwifi)
https://wiki.postmarketos.org/wiki/Qualcomm_Snapdragon_460/662_(SM4250/SM6115)
```

Though pmos doesn't list any active user for T505 itself, but its sibling the gta4lwifi has some work in progress, which is a good sign.
So now we setup the pmbootstrap , which is the central command-line application for postmarketOS development. Among other things, it allows building packages, creating installation images and flashing them to your device.

Installing pmbootstrap from GURU repository in Gentoo:
```bash
sudo emerge -avq dev-util/pmbootstrap
```
pmbootstrap working directory is
```bash
$HOME/.local/var/pmbootstrap/cache_git/pmaports
```
Also I will create another directory in my home, to have easy access to the source files and organising the project directory
```bash
mkdir ~/postmarketos
cd ~/postmarketos
```

```bash
┌[honken@Pratyush-PC] [/dev/pts/4] [main]
└[~/postmarketos]> pmbootstrap init
[19:33:21] Location of the 'work' path. Multiple chroots (native, device arch, device rootfs) will be created in there.
[19:33:21] Work path [/home/honken/.local/var/pmbootstrap]:
[19:33:23] Location of the 'pmaports' path, containing package definitions.
[19:33:23] pmaports path [/home/honken/.local/var/pmbootstrap/cache_git/pmaports]:
[19:33:25] Choose the postmarketOS release channel.
[19:33:25] Available (13):
[19:33:25] * edge: Rolling release / Most devices / Occasional breakage: https://postmarketos.org/edge
[19:33:25] * v25.12: Latest release / Recommended for best stability
[19:33:25] * v25.06: Old release (unsupported)
[19:33:25] Channel [edge]:
[19:33:26] Choose your target device vendor (either an existing one, or a new one for porting).
[19:33:26] Available vendors (107): acer, alcatel, amazon, amediatech, amlogic, apple, ark, arrow, asus, ayaneo, ayn, bananapi, barnesnoble, beelink, blackberry, bq, clockworkpi, cubietech, cutiepi, dell, dongshanpi, epson, essential, fairphone, finepower, fly, fxtec, generic, goclever, google, gp, hisense, hp, htc, huawei, inet, infocus, jolla, khadas, klipad, kobo, lark, leeco, lenovo, lg, librecomputer, linksys, lynx, mangopi, medion, meizu, microsoft, mnt, mobvoi, motorola, nextbit, nobby, nokia, nothing, nvidia, odroid, oneplus, oppo, ouya, pine64, planet, pocketbook, postmarketos, powkiddy, purism, qcom, qemu, qualcomm, radxa, raspberry, realme, rockchip, samsung, semc, sharp, shift, sipeed, solidrun, sony, sourceparts, sqfmi, starway, surftab, t2m, thundercomm, tokio, tolino, trekstor, valve, vernee, vivo, volla, wd, wexler, wiko, wileyfox, xiaomi, xunlong, yu, zhihe, zte, zuk
[19:33:26] Vendor [qemu]: samsung
[19:33:32] Devices are categorised as follows, from best to worst:
* Main: ports where mostly everything works.
* Community: often mostly usable, but may lack important functionality.
* Testing: anything from "just boots in some sense" to almost fully functioning ports.
* Downstream: ports that use a downstream kernel — very limited functionality. Not recommended.

Available devices by codename (42): a32 (downstream), a51 (downstream), chagallwifi (testing), codina (testing), codina-tmo (testing), coreprimevelte (community), e7 (testing), espresso10 (community), espresso7 (community), expressatt (testing), exynos7 (testing), exynos9820 (testing), fortuna (testing), fortunaltezt (testing), gavini (testing), golden (testing), grandmax (testing), gt510 (testing), gt58 (testing), i9100 (testing), j3ltetw (testing), j4lte (downstream), j5 (testing), j5x (testing), janice (testing), jflte (testing), klimtlte (testing), kona (testing), kyle (testing), loganrelte (testing), lt01 (testing), lt03lte (testing), m0 (community), m20lte (downstream), m3 (testing), n1awifi (testing), n2awifi (testing), p4note (testing), rossa (testing), skomer (testing), starqltechn (community), v1awifi (testing)
[19:33:32] Device codename: gta4l
[19:33:46] The specified device ('samsung-gta4l') could not be found in existing ports.
If you're trying to select a device that is supported, check the following:

* Make sure you spelled the vendor name and codename correctly.
* Check if you're on the right release branch. Devices in the 'downstream' category are only available on the 'edge' branch; if you selected a stable branch, run pmbootstrap init again and select 'edge'.
* If the device is supported by a generic port, type in the vendor and codename of the generic port.

If you want to create a new device port, follow the guide at <https://postmarketos.org/porting/>.
[19:33:46] Do you want to create a new device port? (y/n) [y]: y
[19:34:13] What type of port are you creating?
[19:34:13] * mainline: Port using upstream/mainline kernel, compatible with upstream user space.
[19:34:13] * downstream: Port using downstream kernel, using the original (e.g. Android) kernel sources, at least partially incompatible with upstream user space.
[19:34:13] Type? (mainline/downstream): mainline
[19:34:27] Generating new aports for: samsung-gta4l...
[19:34:27] Device architecture (x86/loongarch64/s390x/riscv64/armv7/ppc64le/aarch64/x86_64) [aarch64]:
[19:34:32] Who produced the device (e.g. LG)?
[19:34:32] Manufacturer: samsung
[19:34:38] What is the official name (e.g. Google Nexus 5)?
[19:34:38] Name: Samsung Galaxy Tab A7
[19:34:56] In what year was the device released (e.g. 2012)?
[19:34:56] Year: 2020
[19:35:01] What type of device is it?
[19:35:01] Valid types are: desktop, laptop, convertible, server, tablet, handset, watch, embedded, vm
[19:35:01] Leave empty if this property is defined elsewhere, e.g. in Devicetree or ACPI
[19:35:01] Chassis: tablet
[19:35:12] Does the device have a sdcard or other external storage medium? (y/n) [n]: y
[19:35:17] Which flash method does the device support?
[19:35:17] Flash method (0xffff/fastboot/heimdall/mtkclient/none/rkdeveloptool/uuu) [none]: fastboot
[19:38:10] You can analyze a known working boot.img file to automatically fill out the flasher information for your deviceinfo file. Either specify the path to an image or press return to skip this step (you can do it later with 'pmbootstrap bootimg_analyze').
[19:38:10] Path:
[19:38:35] *** pmaport generated: /home/honken/.local/var/pmbootstrap/cache_git/pmaports/device/testing/device-samsung-gta4l
[19:38:35] Username [user]: nandiprh
[19:39:00] Available providers for postmarketos-base-ui-audio-backend (2):
[19:39:00] * pulseaudio: Use pulseaudio as the audio backend. (default)
[19:39:00] * pipewire: Use pipewire as the audio backend. (but may not work with all devices)
[19:39:00] Provider [default]: pipewire
[19:39:50] Available providers for postmarketos-base-ui-wifi (2):
[19:39:50] * wpa_supplicant: Use wpa_supplicant as the WiFi backend. (default)
[19:39:50] * iwd: Use iwd as the WiFi backend (but may not work with all devices)
[19:39:50] Provider [default]: iwd
[19:40:23] Available providers for postmarketos-usb-moded-default-profile (2):
[19:40:23] * developer: Make 'developer mode' the default usb-moded profile (always enables usb networking) (default)
[19:40:23] * charging: Make 'charging mode' the default usb-moded profile (usb networking must be manually enabled)
[19:40:23] Provider [default]:
[19:40:49] Available user interfaces (17):
[19:40:49] * none: Bare minimum OS image for testing and manual customization. The "console" UI should be selected if a graphical UI is not desired.
[19:40:49] * buffyboard: Plain framebuffer console with modern touchscreen keyboard support
[19:40:49] * console: Console environment, with no graphical/touch UI
[19:40:49] * fbkeyboard: Plain framebuffer console with touchscreen keyboard support
[19:40:49] * gnome: (Wayland) Gnome Shell
[19:40:49] * gnome-mobile: (Wayland) Gnome Shell patched to adapt better to phones (Experimental)
[19:40:49] * i3wm: (X11) Tiling WM (keyboard required)
[19:40:49] * lxqt: (X11) Lightweight Qt Desktop Environment (stylus recommended)
[19:40:49] * mate: (X11) MATE Desktop Environment, fork of GNOME2 (stylus recommended)
[19:40:49] * openbox: (X11) A highly configurable and lightweight X11 window manager (keyboard required)
[19:40:49] * os-installer: UI for installing postmarketOS
[19:40:49] * plasma-bigscreen: (Wayland) 10-feet variant of Plasma, made for big screen TVs
[19:40:49] * plasma-desktop: (Wayland) KDE Desktop Environment (works well with tablets)
[19:40:49] * shelli: Plain console with touchscreen gesture support
[19:40:49] * sxmo-de-dwm: Simple Mobile: Mobile environment based on SXMO and running on dwm
[19:40:49] * sxmo-de-i3: Simple Mobile: Mobile environment based on SXMO and running on i3
[19:40:49] * windowmaker: (X11) Window manager inspired by the NeXTSTEP user interface (stylus recommended)
[19:40:49] * xfce4: (X11) Lightweight desktop (stylus recommended)
[19:40:49] NOTE: 13 UIs are hidden because "deviceinfo_drm" is not set (see https://postmarketos.org/deviceinfo).
[19:40:49] User interface [console]: none
[19:42:58] WARNING: systemd requires kernel version 5.4. Installing systemd with older kernel may result in non-bootable system. Get more information for systemd requirements at https://github.com/systemd/systemd/blob/main/README
[19:42:58] Based on your UI selection, 'default' will result in not installing systemd.
[19:42:58] Install systemd? (default/always/never) [default]: systemd
[19:43:30] ERROR: Input did not pass validation (regex: ^(default|always|never)$). Please try again.
[19:43:30] Install systemd? (default/always/never) [default]: always
[19:43:40] Additional options: extra free space: 0 MB, boot partition size: 256 MB, parallel jobs: 16, ccache per arch: 5G, sudo timer: False, mirror: http://mirror.postmarketos.org/postmarketos/
[19:43:40] Change them? (y/n) [n]: y
[19:44:06] Set extra free space to 0, unless you ran into a 'No space left on device' error. In that case, the size of the rootfs could not be calculated properly on your machine, and we need to add extra free space to make the image big enough to fit the rootfs (pmbootstrap#1904). How much extra free space do you want to add to the image (in MB)?
[19:44:06] Extra space size: 0
[19:45:36] What should be the boot partition size (in MB)?
[19:45:36] Boot size [256]:
[19:45:40] How many jobs should run parallel on this machine, when compiling?
[19:45:40] Jobs [16]:
[19:45:56] We use ccache to speed up building the same code multiple times. How much space should the ccache folder take up per architecture? After init is through, you can check the current usage with 'pmbootstrap stats'. Answer with 0 for infinite.
[19:45:56] Ccache size [5G]:
[19:46:01] pmbootstrap does everything in Alpine Linux chroots, so your host system does not get modified. In order to work with these chroots, pmbootstrap calls 'sudo' internally. For long running operations, it is possible that you'll have to authorize sudo more than once.
[19:46:01] Enable background timer to prevent repeated sudo authorization? (y/n) [n]: y
[19:46:16] Selected mirror: http://mirror.postmarketos.org/postmarketos/
[19:46:16] Change mirror? (y/n) [n]: y
[19:46:23] Download https://postmarketos.org/mirrors.json
[19:46:24] list of available mirrors:
[19:46:24] [1]  mirror.postmarketos.org (Falkenstein, Germany)
[19:46:24] [2]  postmarketos.craftyguy.net (Santa Clara, CA, USA)
[19:46:24] [3]  mirror.sajattack.xyz (Victoria, BC, Canada)
[19:46:24] [4]  mirror.math.princeton.edu (Princeton, NJ, USA)
[19:46:24] [5]  mirrors.aliyun.com (Hangzhou, China)
[19:46:24] [6]  mirrors.tuna.tsinghua.edu.cn (Beijing, China)
[19:46:24] [7]  mirrors.bfsu.edu.cn (Beijing, China)
[19:46:24] [8]  mirrors.ustc.edu.cn (Anhui, China)
[19:46:24] [9]  mirror.nju.edu.cn (Nanjing, Jiangsu, China)
[19:46:24] [10] alpine.sakamoto.pl (Warsaw, Poland)
[19:46:24] [11] distrohub.kyiv.ua (Kyiv, Ukraine)
[19:46:24] [12] mirror-sg.mainlining.org (Singapore, Singapore)
[19:46:24] [13] ftp.halifax.rwth-aachen.de (Aachen, Germany)
[19:46:24] choose 'best' to select the one closest to you
[19:46:24] Select a mirror [1]:
[19:46:37] Additional packages that will be installed to rootfs. Specify them in a comma separated list (e.g.: vim,file) or "none"
[19:46:37] Extra packages [none]: vim,dropbear,nano,strace,evtest,usbutils,tree
[19:48:15] Your host timezone: Asia/Kolkata
[19:48:15] Use this timezone instead of GMT? (y/n) [y]: y
[19:48:20] Choose your preferred locale, like e.g. en_US. Only UTF-8 is supported, it gets appended automatically. Use tab-completion if needed.
[19:48:20] Locale [en_US]:
[19:48:25] Device hostname (short form, e.g. 'foo') [samsung-gta4l]:
[19:48:29] SSH public keys found (2):
[19:48:29] * /home/honken/.ssh/id_ed25519.pub
[19:48:29] * /home/honken/.ssh/id_ed25519.pub.pub
[19:48:29] See https://postmarketos.org/ssh-key-glob for more information.
[19:48:29] Would you like to copy these public keys to the device? (y/n) [n]: y
[19:48:36] After pmaports are changed, the binary packages may be outdated. If you want to install postmarketOS without changes, reply 'n' for a faster installation.
[19:48:36] Build outdated packages during 'pmbootstrap install'? (y/n) [y]: y
[19:49:04] DONE!
```
The above selections are based on my preferences, iwd is modern and integrates better with systemd, I choose pipewire because it was designed with bluetooth in mind from start, and bluetooth is a must for me.
The above extra packages are for basic utilities like vim and nano for text editing, dropbear for a lighweight ssh client, strace syscalls a process makes, highly useful when a daemon fails silently without providing any errors.
```bash
strace daemon_name     # shows what a daemon is doing at kernel level
```
evtest reads raw inputs from /dev/input/event* devices, evtest detects if kernel is receiving any input before any UI layer is involved.

After pmbootstrap init completes, check the status.
```bash
└[~/postmarketos]> pmbootstrap status
Channel: systemd-edge (pmaports: main, dirty)
Device:  samsung-gta4l (aarch64)
UI:      none
systemd: yes ('always' selected in 'pmbootstrap init')
```
Then check if the device info is intact, and hasn't undergone any accidental changes.
```bash
└[~/postmarketos]> cat ~/.local/var/pmbootstrap/cache_git/pmaports/device/testing/device-samsung-gta4l/deviceinfo
# Reference: <https://postmarketos.org/deviceinfo>
# Please use double quotes only. You can source this file in shell
# scripts.

deviceinfo_format_version="0"
deviceinfo_name="samsung Samsung Galaxy Tab A7"
deviceinfo_manufacturer="samsung"
deviceinfo_codename="samsung-gta4l"
deviceinfo_year="2020"
deviceinfo_dtb=""
deviceinfo_arch="aarch64"

# Device related
deviceinfo_chassis="tablet"
deviceinfo_external_storage="true"

# Bootloader related
deviceinfo_flash_method="fastboot"
deviceinfo_kernel_cmdline=""
deviceinfo_generate_bootimg="true"
deviceinfo_flash_pagesize="2048"
deviceinfo_bootimg_qcdt="false"
deviceinfo_dtb_second="false"
```
For now this looks fine to me, also above I choose fastboot as flashing method rather than heimdall because my device currently runs lineage os, so I do have access to userspace level fastbootd, which is much simpler and faster. 
Though in most cases heimdall should be preferred because its intended to be used for samsung devices,as samsung devices use their own odin protocol.

So here we are done with the initial setup, and we can now setup the working directory to clone the mainline kernel, install any required packages, inspect the device partitions and extract the dts files. Though for my device I can simply refer to the LineageOS tree, I will include every option that might be useful for future use cases.
Now we need is adb and the device to be ported need to have debugging enabled in developer options. Also great to have rooted usb debugging to be enabled, by default LineageOS ships with this option enabled in its kernel.

Now we can extract the dtbo image
```bash
[~]> adb devices
* daemon not running; starting now at tcp:5037
* daemon started successfully
List of devices attached
R9ZR502N5LJ     unauthorized

┌[honken@Pratyush-PC] [/dev/pts/3]
└[~]> adb root
restarting adbd as root
┌[honken@Pratyush-PC] [/dev/pts/3]
└[~]> cd postmarketos
┌[honken@Pratyush-PC] [/dev/pts/3] [main]
└[~/postmarketos]> adb pull /sys/firmware/fdt ~/postmarketos/gta4l_extracted.dtb
/sys/firmware/fdt: 1 file pulled, 0 skipped. 20.0 MB/s (817870 bytes in 0.039s)
┌[honken@Pratyush-PC] [/dev/pts/3] [main ⚡]
└[~/postmarketos]> ls
gta4l_extracted.dtb


# If adb root is unavailable, make use of 
$ adb shell su -c "cat /sys/firmware/fdt" > gta4l_extracted.dtb
```
Currently we have the merged DTB the bootloader passed to the kernel at boot. Base SoC dtb and device specific dtbo overlay applied, hence the complete hardware description.
As earlier stated we need is the DTS file, but we do have the binary generated from it ,the dtc (Open Firmware device tree compiler).

```bash
sudo emerge -av sys-apps/dtc

└[~/postmarketos]> dtc -I dtb -O dts -o ~/postmarketos/gta4l_extracted.dts ~/postmarketos/gta4l_extracted.dtb
/home/honken/postmarketos/gta4l_extracted.dts: Warning (label_is_string): /soc/qcom,cam_smmu/msm_cam_smmu_ope:label: property is not a string
/home/honken/postmarketos/gta4l_extracted.dts: Warning (reg_format): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l1@4000:reg: property has invalid length (4 bytes) (#address-cells == 2, #size-cells == 1)
/home/honken/postmarketos/gta4l_extracted.dts: Warning (reg_format): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l2@4100:reg: property has invalid length (4 bytes) (#address-cells == 2, #size-cells == 1)
/home/honken/postmarketos/gta4l_extracted.dts: Warning (reg_format): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l3@4200:reg: property has invalid length (4 bytes) (#address-cells == 2, #size-cells == 1)
/home/honken/postmarketos/gta4l_extracted.dts: Warning (reg_format): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l4@4300:reg: property has invalid length (4 bytes) (#address-cells == 2, #size-cells == 1)
/home/honken/postmarketos/gta4l_extracted.dts: Warning (reg_format): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l5@4400:reg: property has invalid length (4 bytes) (#address-cells == 2, #size-cells == 1)
/home/honken/postmarketos/gta4l_extracted.dts: Warning (reg_format): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l6@4400:reg: property has invalid length (4 bytes) (#address-cells == 2, #size-cells == 1)
/home/honken/postmarketos/gta4l_extracted.dts: Warning (reg_format): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l7@4400:reg: property has invalid length (4 bytes) (#address-cells == 2, #size-cells == 1)
/home/honken/postmarketos/gta4l_extracted.dts: Warning (reg_format): /soc/qcom,msm-audio-apr/qcom,q6core-audio/lpi_pinctrl@ac40000:reg: property has invalid length (8 bytes) (#address-cells == 2, #size-cells == 1)
/home/honken/postmarketos/gta4l_extracted.dts: Warning (reg_format): /soc/qcom,msm-audio-apr/qcom,q6core-audio/bolero-cdc/va-macro@a730000:reg: property has invalid length (8 bytes) (#address-cells == 2, #size-cells == 1)
/home/honken/postmarketos/gta4l_extracted.dts: Warning (reg_format): /soc/qcom,msm-audio-apr/qcom,q6core-audio/bolero-cdc/rx-macro@a600000:reg: property has invalid length (8 bytes) (#address-cells == 2, #size-cells == 1)
/home/honken/postmarketos/gta4l_extracted.dts: Warning (reg_format): /soc/qcom,msm-audio-apr/qcom,q6core-audio/bolero-cdc/tx-macro@a620000:reg: property has invalid length (8 bytes) (#address-cells == 2, #size-cells == 1)
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /memory: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/wake-gic: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,cpufreq-hw: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pm6125@0/vadc@3100/ref_gnd: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pm6125@0/vadc@3100/vref_1p25: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pm6125@0/vadc@3100/die_temp: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pm6125@0/vadc@3100/vph_pwr: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pm6125@0/vadc@3100/vcoin: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pm6125@0/vadc@3100/xo_therm: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pm6125@0/vadc@3100/pa_therm0: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pm6125@0/vadc@3100/quiet_therm: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pm6125@0/vadc@3100/camera_flash_therm: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pm6125@0/vadc@3100/emmc_ufs_therm: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pm6125@0/adc_tm@3500/pa_therm0: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pm6125@0/adc_tm@3500/quiet_therm: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pm6125@0/adc_tm@3500/xo_therm: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pm6125@0/adc_tm@3400/camera_flash_therm: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pm6125@0/adc_tm@3400/emmc_ufs_therm: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/ref_gnd: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/vref_1p25: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/die_temp: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/vph_pwr: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/vbat_sns: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/usb_in_i_uv: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/usb_in_v_div_16: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/chg_temp: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/bat_therm: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/bat_therm_30k: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/bat_therm_400k: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/bat_id: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/i_parallel: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/v_i_int_ext: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/v_i_parallel: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/conn_therm: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/vadc@3100/skin_therm: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@2/adc_tm@3500/skin_therm: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@3/qcom,pwms@b300/lpg@1: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@3/qcom,pwms@b300/lpg@2: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,spmi@1c40000/qcom,pmi632@3/qcom,pwms@b300/lpg@3: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/tpdm@8a26000: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/tpdm@899c000: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/ad-hoc-bus: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/spi@4a80000/novatek@0/novatek-mp-criteria-720C@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/spi@4a80000/novatek@0/novatek-mp-criteria-720E@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/spi@4a80000/novatek@0/novatek-mp-criteria-720F@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/spi@4a80000/novatek@0/novatek-mp-criteria-7215@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,kgsl-3d0@5900000/qcom,gpu-cx-ipeak/qcom,gpu-cx-ipeak@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,kgsl-3d0@5900000/qcom,gpu-cx-ipeak/qcom,gpu-cx-ipeak@1: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,msm-audio-apr/qcom,q6core-audio/msm_cdc_pinctrl@18: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,msm-audio-apr/qcom,q6core-audio/bolero-cdc/va-macro@a730000/va_swr_master/wcd937x-tx-slave: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,msm-audio-apr/qcom,q6core-audio/bolero-cdc/rx-macro@a600000/rx_swr_master/wcd937x-rx-slave: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_dsi0_pll: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_td4330_truly_v2_cmd/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_td4330_truly_v2_cmd/qcom,mdss-dsi-display-timings/timing@1: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_td4330_truly_v2_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_td4330_truly_v2_video/qcom,mdss-dsi-display-timings/timing@1: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_nt36525_truly_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_r66451_hd_plus_90hz_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_r66451_hd_plus_90hz_cmd/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_hx83102_inx_fhd_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_sim_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_hx83102e_inx_fhd_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_hx83102e_txd_inx_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_hx83102e_hlt_auo_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_hx83102e_txd_auo_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_nt36523_lce_panda_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_nt36523_hlt_auo_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_hx83102e_txd_auo_al_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_hx83102e_lide_hsd_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_nt36523_lide_hsd_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_nt36523_txd_inx_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_ft8201ab_lide_hsd_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_hx83102e_tianma_tianma_al_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_hx83102e_djn_jdi_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_mdp/qcom,mdss_dsi_ft8201ab_tianma_tianma_video/qcom,mdss-dsi-display-timings/timing@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_rotator: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_dsi0_ctrl: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,mdss_dsi_phy0: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,csiphy0: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,csiphy1: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,csiphy2: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,cci0: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/msm_cdc_pinctrl@92: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/msm_cdc_pinctrl@106: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,wb-display@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,msm_notifier@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,camera-flash@0: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,camera-flash@1: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/qcom,camera-flash@2: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/sn_fuse: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /soc/sec_boot_fuse: node has a reg or ranges property, but no unit name
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@1: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@1/inputbooster,resource/resource@1: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@1/inputbooster,resource/resource@2: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@1/inputbooster,resource/resource@3: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@1/inputbooster,resource/resource@4: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@2: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@2/inputbooster,resource/resource@1: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@2/inputbooster,resource/resource@2: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@2/inputbooster,resource/resource@3: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@2/inputbooster,resource/resource@4: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@3: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@3/inputbooster,resource/resource@1: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@3/inputbooster,resource/resource@2: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@3/inputbooster,resource/resource@3: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@3/inputbooster,resource/resource@4: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@4: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@4/inputbooster,resource/resource@1: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@4/inputbooster,resource/resource@2: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@4/inputbooster,resource/resource@3: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@4/inputbooster,resource/resource@4: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@5: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@5/inputbooster,resource/resource@1: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@5/inputbooster,resource/resource@2: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@5/inputbooster,resource/resource@3: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@5/inputbooster,resource/resource@4: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@6: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@6/inputbooster,resource/resource@1: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@6/inputbooster,resource/resource@2: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@6/inputbooster,resource/resource@3: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@6/inputbooster,resource/resource@4: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@7: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@7/inputbooster,resource/resource@1: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@7/inputbooster,resource/resource@2: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@7/inputbooster,resource/resource@3: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@7/inputbooster,resource/resource@4: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@8: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@8/inputbooster,resource/resource@1: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@8/inputbooster,resource/resource@2: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@8/inputbooster,resource/resource@3: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@8/inputbooster,resource/resource@4: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@9: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@9/inputbooster,resource/resource@1: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@9/inputbooster,resource/resource@2: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@9/inputbooster,resource/resource@3: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_vs_reg): /input_booster/booster_key@9/inputbooster,resource/resource@4: node has a unit name, but no reg or ranges property
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_format): /soc/ssusb@4e00000/qcom,usbbam@0x04f04000: unit name should not have leading "0x"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_format): /soc/ssusb@4e00000/qcom,usbbam@0x04f04000: unit name should not have leading 0s
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_format): /soc/kgsl-smmu@0x59a0000/gfx_0_tbu@0x59c5000: unit name should not have leading "0x"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_format): /soc/apps-smmu@0xc600000/anoc_1_tbu@0xc785000: unit name should not have leading "0x"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_format): /soc/apps-smmu@0xc600000/mm_rt_tbu@0xc789000: unit name should not have leading "0x"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_format): /soc/apps-smmu@0xc600000/mm_nrt_tbu@0xc78d000: unit name should not have leading "0x"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unit_address_format): /soc/apps-smmu@0xc600000/cdsp_tbu@0xc791000: unit name should not have leading "0x"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (pci_device_reg): Failed prerequisite 'reg_format'
/home/honken/postmarketos/gta4l_extracted.dts: Warning (pci_device_bus_num): Failed prerequisite 'reg_format'
/home/honken/postmarketos/gta4l_extracted.dts: Warning (simple_bus_reg): Failed prerequisite 'reg_format'
/home/honken/postmarketos/gta4l_extracted.dts: Warning (i2c_bus_reg): Failed prerequisite 'reg_format'
/home/honken/postmarketos/gta4l_extracted.dts: Warning (spi_bus_reg): Failed prerequisite 'reg_format'
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l1@4000: Relying on default #address-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l1@4000: Relying on default #size-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l2@4100: Relying on default #address-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l2@4100: Relying on default #size-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l3@4200: Relying on default #address-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l3@4200: Relying on default #size-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l4@4300: Relying on default #address-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l4@4300: Relying on default #size-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l5@4400: Relying on default #address-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l5@4400: Relying on default #size-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l6@4400: Relying on default #address-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l6@4400: Relying on default #size-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l7@4400: Relying on default #address-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/i2c@4a84000/qcom,pm8008@9/qcom,pm8008-regulator/qcom,pm8008-l7@4400: Relying on default #size-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/qcom,msm-audio-apr/qcom,q6core-audio/lpi_pinctrl@ac40000: Relying on default #address-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/qcom,msm-audio-apr/qcom,q6core-audio/lpi_pinctrl@ac40000: Relying on default #size-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/qcom,msm-audio-apr/qcom,q6core-audio/bolero-cdc/va-macro@a730000: Relying on default #address-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/qcom,msm-audio-apr/qcom,q6core-audio/bolero-cdc/va-macro@a730000: Relying on default #size-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/qcom,msm-audio-apr/qcom,q6core-audio/bolero-cdc/rx-macro@a600000: Relying on default #address-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/qcom,msm-audio-apr/qcom,q6core-audio/bolero-cdc/rx-macro@a600000: Relying on default #size-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/qcom,msm-audio-apr/qcom,q6core-audio/bolero-cdc/tx-macro@a620000: Relying on default #address-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_default_addr_size): /soc/qcom,msm-audio-apr/qcom,q6core-audio/bolero-cdc/tx-macro@a620000: Relying on default #size-cells value
/home/honken/postmarketos/gta4l_extracted.dts: Warning (avoid_unnecessary_addr_size): Failed prerequisite 'avoid_default_addr_size'
/home/honken/postmarketos/gta4l_extracted.dts: Warning (unique_unit_address): Failed prerequisite 'avoid_default_addr_size'
/home/honken/postmarketos/gta4l_extracted.dts: Warning (gpios_property): /soc/i2c@4a84000/wsa881x-i2c-codec@e: Missing property '#gpio-cells' in node /soc/qcom,msm-audio-apr/qcom,q6core-audio/msm_cdc_pinctrl@18 or bad phandle (referred from qcom,wsa-analog-clk-gpio[0])
/home/honken/postmarketos/gta4l_extracted.dts: Warning (gpios_property): /soc/i2c@4a84000/wsa881x-i2c-codec@e: Missing property '#gpio-cells' in node /soc/msm_cdc_pinctrl@106 or bad phandle (referred from qcom,wsa-analog-reset-gpio[0])
/home/honken/postmarketos/gta4l_extracted.dts: Warning (gpios_property): /soc/qcom,msm-audio-apr/qcom,q6core-audio/lpi_pinctrl@ac40000: Missing property '#gpio-cells' in node /soc/qcom,rpm-smd/rpm-regulator-ldoa17/regulator-l17 or bad phandle (referred from qcom,num-gpios[0])
/home/honken/postmarketos/gta4l_extracted.dts: Warning (gpios_property): /soc/qcom,msm-audio-apr/qcom,q6core-audio/bolero-cdc/va-macro@a730000: Missing property '#gpio-cells' in node /soc/wake-gic or bad phandle (referred from qcom,is-used-swr-gpio[0])
/home/honken/postmarketos/gta4l_extracted.dts: Warning (gpios_property): /soc/qcom,msm-audio-apr/qcom,q6core-audio/bolero-cdc/va-macro@a730000: Missing property '#gpio-cells' in node /soc/qcom,msm-audio-apr/qcom,q6core-audio/va_swr_clk_data_pinctrl or bad phandle (referred from qcom,va-swr-gpios[0])
/home/honken/postmarketos/gta4l_extracted.dts: Warning (gpios_property): /soc/qcom,msm-audio-apr/qcom,q6core-audio/bolero-cdc/rx-macro@a600000: Missing property '#gpio-cells' in node /soc/qcom,msm-audio-apr/qcom,q6core-audio/rx_swr_clk_data_pinctrl or bad phandle (referred from qcom,rx-swr-gpios[0])
/home/honken/postmarketos/gta4l_extracted.dts: Warning (gpios_property): /soc/qcom,msm-audio-apr/qcom,q6core-audio/sound: Missing property '#gpio-cells' in node /soc/msm_cdc_pinctrl_pri or bad phandle (referred from qcom,pri-mi2s-gpios[0])
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@9800000/ports/port@1: graph node unit address error, expected "0"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@9800000/ports/port@2: graph node unit address error, expected "1"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@9800000/ports/port@3: graph node unit address error, expected "2"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@9800000/ports/port@4: graph node unit address error, expected "3"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@9800000/ports/port@5: graph node unit address error, expected "4"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@9800000/ports/port@6: graph node unit address error, expected "5"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@9800000/ports/port@7: graph node unit address error, expected "6"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@9800000/ports/port@8: graph node unit address error, expected "7"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/tpda@9832000/ports/port@1: graph node unit address error, expected "0"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/tpda@9862000/ports/port@1: graph node unit address error, expected "0"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/tpda@98c0000/ports/port@1: graph node unit address error, expected "0"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@9810000/ports/port@1: graph node unit address error, expected "0"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/tpda@8a04000/ports/port@1: graph node unit address error, expected "0"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@8944000/ports/port@1: graph node unit address error, expected "0"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@8861000/ports/port@2: graph node unit address error, expected "0"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@8861000/ports/port@3: graph node unit address error, expected "1"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/tpda@8004000/ports/port@1: graph node unit address error, expected "0"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/tpda@8004000/ports/port@2: graph node unit address error, expected "1"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/tpda@8004000/ports/port@3: graph node unit address error, expected "5"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/tpda@8004000/ports/port@4: graph node unit address error, expected "7"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/tpda@8004000/ports/port@5: graph node unit address error, expected "8"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/tpda@8004000/ports/port@6: graph node unit address error, expected "a"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/tpda@8004000/ports/port@7: graph node unit address error, expected "c"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/tpda@8004000/ports/port@8: graph node unit address error, expected "d"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/tpda@8004000/ports/port@9: graph node unit address error, expected "f"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@8005000/ports/port@1: graph node unit address error, expected "0"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@8005000/ports/port@2: graph node unit address error, expected "6"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@8005000/ports/port@3: graph node unit address error, expected "5"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@8005000/ports/port@4: graph node unit address error, expected "5"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@8041000/ports/port@1: graph node unit address error, expected "5"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@8041000/ports/port@2: graph node unit address error, expected "6"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@8041000/ports/port@3: graph node unit address error, expected "7"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@8042000/ports/port@5: graph node unit address error, expected "6"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@8045000/ports/port@1: graph node unit address error, expected "0"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/funnel@8045000/ports/port@2: graph node unit address error, expected "1"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/tmc@8047000/ports/port@1: graph node unit address error, expected "0"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_port): /soc/replicator@8046000/ports/port@1: graph node unit address error, expected "0"
/home/honken/postmarketos/gta4l_extracted.dts: Warning (graph_child_address): Failed prerequisite 'graph_port'
┌[honken@Pratyush-PC] [/dev/pts/3] [main ⚡]
└[~/postmarketos]> ls
gta4l_extracted.dtb  gta4l_extracted.dts
```

```bash
└[~/postmarketos-old]> ls ~/.local/var/pmbootstrap/cache_git/pmaports/device/testing/device-samsung-gta4l/

# Should contain:
# APKBUILD
# deviceinfo
APKBUILD  deviceinfo  modules-initfs
┌[honken@Pratyush-PC] [/dev/pts/1]
└[~/postmarketos-old]> cat ~/.local/var/pmbootstrap/cache_git/pmaports/device/testing/device-samsung-gta4l/deviceinfo ; cat ~/.local/var/pmbootstrap/cache_git/pmaports/device/testing/device-samsung-gta4l/APKBUILD
# Reference: <https://postmarketos.org/deviceinfo>
# Please use double quotes only. You can source this file in shell
# scripts.

deviceinfo_format_version="0"
deviceinfo_name="samsung Samsung Galaxy Tab A7"
deviceinfo_manufacturer="samsung"
deviceinfo_codename="samsung-gta4l"
deviceinfo_year="2020"
deviceinfo_dtb=""
deviceinfo_arch="aarch64"

# Device related
deviceinfo_chassis="tablet"
deviceinfo_external_storage="true"

# Bootloader related
deviceinfo_flash_method="fastboot"
deviceinfo_kernel_cmdline=""
deviceinfo_generate_bootimg="true"
deviceinfo_flash_pagesize="2048"
deviceinfo_bootimg_qcdt="false"
deviceinfo_dtb_second="false"
# Reference: <https://postmarketos.org/devicepkg>
maintainer=""
pkgname=device-samsung-gta4l
pkgdesc="samsung Samsung Galaxy Tab A7"
pkgver=1
pkgrel=0
url="https://postmarketos.org"
license="MIT"
arch="aarch64"
options="!check !archcheck"
depends="
        linux-CHANGEME
        mkbootimg
        postmarketos-base
"
makedepends="devicepkg-dev"
source="
        deviceinfo
        modules-initfs
"

build() {
        devicepkg_build $startdir $pkgname
}

package() {
        devicepkg_package $startdir $pkgname
}

sha512sums="(run 'pmbootstrap checksum device-samsung-gta4l' to fill)"
```

We shall now get a look of the partition table
```bash
└[~/postmarketos]> fastboot getvar all
(bootloader) battery-part-status:unsupported
(bootloader) cpu-abi:arm64-v8a
(bootloader) super-partition-name:super
(bootloader) battery-voltage:4364
(bootloader) is-force-debuggable:no
(bootloader) treble-enabled:true
(bootloader) is-userspace:yes
(bootloader) max-fetch-size:0x10000000
(bootloader) partition-size:userdata:0x577C7BE00
(bootloader) partition-size:cache:0xC800000
(bootloader) partition-size:rpm:0x80000
(bootloader) partition-size:storsec:0x20000
(bootloader) partition-size:mdtp:0x2000000
(bootloader) partition-size:ssd:0x40000
(bootloader) partition-size:mmcblk0:0x747C00000
(bootloader) partition-size:recovery:0x62C0000
(bootloader) partition-size:misc:0x100000
(bootloader) partition-size:vbmeta:0x10000
(bootloader) partition-size:super:0x157000000
(bootloader) partition-size:metadata:0x1000000
(bootloader) partition-size:dtbo:0x1800000
(bootloader) partition-size:boot:0x6000000
(bootloader) partition-size:system:0x5F4C9000
(bootloader) partition-size:vendor:0x213F0000
(bootloader) partition-size:product:0x97757000
(bootloader) partition-size:odm:0x1A0000
(bootloader) version-vndk:
(bootloader) partition-type:userdata:raw
(bootloader) partition-type:cache:raw
(bootloader) partition-type:rpm:raw
(bootloader) partition-type:storsec:raw
(bootloader) partition-type:mdtp:raw
(bootloader) partition-type:ssd:raw
(bootloader) partition-type:mmcblk0:raw
(bootloader) partition-type:recovery:raw
(bootloader) partition-type:misc:raw
(bootloader) partition-type:vbmeta:raw
(bootloader) partition-type:super:raw
(bootloader) partition-type:metadata:raw
(bootloader) partition-type:dtbo:raw
(bootloader) partition-type:boot:raw
(bootloader) partition-type:system:raw
(bootloader) partition-type:vendor:raw
(bootloader) partition-type:product:raw
(bootloader) partition-type:odm:raw
(bootloader) battery-serial-number:unsupported
(bootloader) has-slot:userdata:no
(bootloader) has-slot:cache:no
(bootloader) has-slot:rpm:no
(bootloader) has-slot:storsec:no
(bootloader) has-slot:mdtp:no
(bootloader) has-slot:ssd:no
(bootloader) has-slot:mmcblk0:no
(bootloader) has-slot:recovery:no
(bootloader) has-slot:misc:no
(bootloader) has-slot:vbmeta:no
(bootloader) has-slot:super:no
(bootloader) has-slot:metadata:no
(bootloader) has-slot:dtbo:no
(bootloader) has-slot:boot:no
(bootloader) has-slot:system:no
(bootloader) has-slot:vendor:no
(bootloader) has-slot:product:no
(bootloader) has-slot:odm:no
(bootloader) security-patch-level:2026-07-01
(bootloader) vendor-fingerprint:samsung/gta4leea/gta4l:12/SP1A.210812.016/T505XXS8CXG1:user/release-keys
(bootloader) hw-revision:0
(bootloader) current-slot:
(bootloader) serialno:R9ZR502N5LJ
(bootloader) product:gta4l
(bootloader) version-os:16
(bootloader) first-api-level:29
(bootloader) slot-count:0
(bootloader) max-download-size:0x10000000
(bootloader) version:0.4
(bootloader) version-baseband:
(bootloader) is-logical:userdata:no
(bootloader) is-logical:cache:no
(bootloader) is-logical:rpm:no
(bootloader) is-logical:storsec:no
(bootloader) is-logical:mdtp:no
(bootloader) is-logical:ssd:no
(bootloader) is-logical:mmcblk0:no
(bootloader) is-logical:recovery:no
(bootloader) is-logical:misc:no
(bootloader) is-logical:vbmeta:no
(bootloader) is-logical:super:no
(bootloader) is-logical:metadata:no
(bootloader) is-logical:dtbo:no
(bootloader) is-logical:boot:no
(bootloader) is-logical:system:yes
(bootloader) is-logical:vendor:yes
(bootloader) is-logical:product:yes
(bootloader) is-logical:odm:yes
(bootloader) battery-soc:100
(bootloader) secure:yes
(bootloader) dynamic-partition:true
(bootloader) system-fingerprint:samsung/gta4leea/gta4l:12/SP1A.210812.016/T505XXS8CXG1:user/release-keys
(bootloader) version-bootloader:T505XXS8CXG1
(bootloader) unlocked:yes
all:
Finished. Total time: 0.278s
```
I'm not pasting the heimdall print-pit output, because its not viable here, but we will extract it and it will remain a good source of comparison
```bash
heimdall print-pit
```

Now we need to clone some important repos, for reference, the device tree from lineage and the android kernel common to the snapdragon 6115.
```bash
git clone https://github.com/LineageOS/android_device_samsung_gta4l
git clone https://github.com/LineageOS/android_kernel_samsung_sm6115
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git --depth=1 --no-checkout

# Also as this a work spanning across months, its reccommended to run "git pull" once a while to ensure we are working with the latest developments.

```

I would end the part-I with referencing fastboot, adb and heimdall sources, also providing some essential commands.

```txt
Official docs:
https://developer.android.com/tools/adb

Source:
https://android.googlesource.com/platform/packages/modules/adb
Official docs:
https://developer.android.com/tools/fastboot

Protocol spec:
https://android.googlesource.com/platform/system/core/+/refs/heads/main/fastboot/README.md

Fastbootd (userspace fastboot) specifically:
https://source.android.com/docs/core/architecture/bootloader/fastbootd

Official repo (Henrik Grimler's maintained fork):
https://gitlab.com/BenjaminDobell/Heimdall

reference:
https://gitlab.com/BenjaminDobell/Heimdall/-/blob/master/README.md 

Flashing guide:
https://wiki.postmarketos.org/wiki/Flashing

Samsung specific:
https://wiki.postmarketos.org/wiki/Samsung_Galaxy_flashing

Samsung Odin protocol (Heimdall is its open implementation):
https://wiki.postmarketos.org/wiki/Odin

PIT file format:
https://wiki.postmarketos.org/wiki/Partition_Information_Table

```
Android protocol, runs from recovery (fastbootd) requires LineageOS recovery to be present, Samsung removes it from their vendor distributed/ stock ROM
Fastboot operations-(top choice)
```bash
fastboot getvar all          # reading partition table
fastboot flash boot boot.img    # flashing boot image
fastboot flash recovery recovery.img  # flashing recovery image
fastboot devices  # device detection
fastboot reboot
fastboot reboot recovery
fastboot reboot bootloader    # reboots to download mode 
fastboot get_staged dtbo.img   # needs inquiry from my end
fastboot flash --read dtbo.img  # needs inquiry
```
Fastboot doesn't support reading partitions(This oeration is like extracting images and not to be confused with reading partition table, which gives the info of disk structure)

Heimdall is reverse engineered based on Samsung's Odin protocol, Download mode, always available at hardware level, independent of installed OS
Heimdall operations
```bash
# Needs to be in download mode, or else heimdall operations don't work.
heimdall print-pit   # print partition-table
heimdall flash --BOOT boot.img   # flash boot partition
heimdall flash --RECOVERY recovery.img    # recovery partition
heimdall download --BOOT lineage_boot.img   # extracting a partition
heimdall download --DTBO dtbo.img   # reading dtb partition, heimdall cannot extract dtb from a live running kernel, only adb can do so
heimdall detect   # detect devices
```

Android Debug Bridge, normal boot or recovery requires USB debugging enabled or recovery ADB
```bash
adb root
adb pull /dev/block/by-name/boot lineage_boot.img

# adb with su
adb shell su -c "dd if=/dev/block/by-name/boot of=/sdcard/boot.img"
adb pull /sdcard/boot.img lineage_boot.img

# extracting dtb fom live running kernel
adb pull /sys/firmware/fdt gta4l_extracted.dtb

# ADB with root
adb shell su -c "dd if=/dev/block/by-name/DTBO of=/sdcard/dtbo.img"
adb pull /sdcard/dtbo.img .

adb devices # device detection
```

With this we have setup the toolchain required, and this will mark the end of Part-I from my end.
I'm planning to actually write next on the detection of board configuration from the gta4l tree. Then gather more sources on dts writing conventions of android and linux.
After that we will start writing the dts file for mainline linux kernel, for which we will consider a reference device listed on linus branch, as there are devices that make use of 6115.dts

My idea is to divide the components or peripherals in a hierarchy/priority
p1.Critical-like ufs storage, serial console and usb
p2.usability-touchscreen, display, power keys,..
p3.connectivity-wifi,bluetooth,modem,..
p4.supporting cast-regulators,cast,..

Well its already too long,I shall close it here.

Thankyou !
