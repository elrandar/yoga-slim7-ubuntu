# Linux on Lenovo Yoga Slim 7 AMD

## About

This are various tweaks and fix to run Linux on Lenovo Yoga Slim 7. This note is validated on the following configuration
- Ubuntu 20.04.1
- Lenovo Yoga Slim 7 AMD (Ryzen 7)

## Summary

Legend:
- :heavy_check_mark: Works out of the box (Kernel 5.4)
- :hammer_and_wrench: Require tweaking
- :negative_squared_cross_mark: Not working
- :question: Unknown

| Feature                      | Status              | Description                                                                         |
| ---------------------------- | ------------------- | ----------------------------------------------------------------------------------- |
| Power (battery and charging) | :heavy_check_mark:  |                                                                                     |
| Storage                      | :heavy_check_mark:  | Disable bitlocker on windows to access windows partition from Linux                 |
| Graphic                      | :hammer_and_wrench: | Kernel update is required (see [below](#Graphic))                                   |
| USB                          | :heavy_check_mark:  |                                                                                     |
| Keyboard                     | :heavy_check_mark:  |                                                                                     |
| Speakers                     | :heavy_check_mark:  | Should work on older software but broken on up to date system (see [below](#Audio)) |
| Microphone                   | :heavy_check_mark:  | It seems there's a bug on kernel 5.7, please use other kernel version               |
| Audio jack                   | :heavy_check_mark:  |                                                                                     |
| Wifi and Bluetooth           | :heavy_check_mark:  |                                                                                     |
| Webcam                       | :heavy_check_mark:  |                                                                                     |
| External display (HDMI)      | :hammer_and_wrench: | Kernel update is required (see [below](#Graphic))                                   |
| Suspend                      | :hammer_and_wrench: | See detail [below](#Suspend)                                                        |

## Fixes

**DISCLAIMER** I am not responsible for any damage and negative consequences to your system

### Graphic

By default Ubuntu 20.04.1 shipped with Linux 5.4, support for AMD 4000 graphics is still experimental on 5.4. To get the best result wait for Ubuntu 20.04.2 or upgrade manually to the latest stable kernel (5.8 by the time of publication).

Some of the ways to upgrade the kernel:
- https://github.com/pimlie/ubuntu-mainline-kernel.sh
- https://github.com/bkw777/mainline

**Note: If you encounter `error: /vmlinux-<version number> has invalid signature.` on boot, you need to disable secure boot on the UEFI setting or follow https://gist.github.com/maxried/796d1f3101b3a03ca153fa09d3af8a11**

### Suspend

#### Background

In recent times Microsoft has introduced something called "[Modern Standby](https://docs.microsoft.com/en-us/windows-hardware/design/device-experiences/modern-standby)" which is essentially a new way to suspend with the advantage of allowing the system to do some task while suspending (e.g. fetching emails). In order to support this mode, the BIOS must not advertise support for the traditional suspend (S3) <sup>[Citation needed]</sup> 

```bash
$ dmesg | grep ACPI:\ \(
[    0.383096] ACPI: (supports S0 S4 S5)

```

By not advertising support for S3, the kernel will only support s2idle sleep mode which is also supported by Linux

```bash
$ cat /sys/power/mem_sleep 
[s2idle]
```

However there seems to be a problem with the amdgpu driver on resuming from suspend in this mode.

```
amdgpu 0000:03:00.0: [drm:amdgpu_ring_test_helper [amdgpu]] *ERROR* ring gfx test failed (-110)
[drm:amdgpu_device_ip_resume_phase2 [amdgpu]] *ERROR* resume of IP block <gfx_v9_0> failed -110
[drm:amdgpu_device_resume [amdgpu]] *ERROR* amdgpu_device_ip_resume failed (-110).
```

#### Possible Solutions

There are three solutions:
- Wait until the problem in amdgpu driver is fixed in newer kernel version
- Wait until Lenovo adds an option in the UEFI to advertise S3 support (similar to the options available in Thinkpad)
- Modify the system to advertise S3 support (see below)

#### Advertise S3

**0. Important notes**

All of the following commands assume **root shell** (sudo -i)

```bash
sudo -i
```

**1. Get the required tools**

Download and compile from https://www.acpica.org/downloads
```bash
# Install some required dependencies
apt install m4 build-essential bison flex

# Download and extract
wget https://acpica.org/sites/acpica/files/acpica-unix-20200717.tar.gz
tar -xvf acpica-unix-20200717.tar.gz
cd acpica-unix-20200717

# Make
make clean
make

# Add to PATH for easy access
PATH=$PATH:$(realpath ./generate/unix/bin/)
```

**2. Dump the ACPI files and decompile the DSDT table**

```bash
# Create new working directory
mkdir acpi
cd acpi

# Dump acpi table to binary files
acpidump -b

# Decompile dsdl.bat with all of the .dat as external symbol
iasl -e *.dat -d dsdt.dat
```

**3. Apply patch**

Copy dsdt.patch from this repo and patch dsdt.dsl
```bash
patch <dsdt.patch
```

**4. Recompile the modified table**

```bash
# Recompile dsdt to new hex asml table (ignore warning)
iasl -ve -tc dsdt.dsl
```

**5. Make override archive**
```bash
mkdir -p kernel/firmware/acpi
cp dsdt.aml kernel/firmware/acpi
find kernel | cpio -H newc --create > acpi_s3_override

# Copy to /boot
cp acpi_s3_override /boot/
```

**6. Set the default sleep type to S3 (deep)**

Open `etc/default/grub` and add `mem_sleep_default=deep` to `GRUB_CMDLINE_LINUX_DEFAULT` then run `update-grub`

Example:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash mem_sleep_default=deep"
```

**7. Set grub to use the override**

**Note: There's a [problem](grub-fix.md) in older version of grub shipped with Ubuntu, make sure you upgrade your system (`apt update && apt upgrade`) before performing this step** 

Open `etc/default/grub` and add `acpi_s3_override` to `GRUB_EARLY_INITRD_LINUX_CUSTOM` then run `update-grub`

Example:
```
GRUB_EARLY_INITRD_LINUX_CUSTOM="acpi_s3_override"
```

**8. Secure boot**

If you are using mainline kernel, skip this step.

If you are using the official kernel (for example in Ubuntu 20.10) be aware that you need to disable secure boot because of the default behaviour of Ubuntu kernel (relevant discussion [#10](https://github.com/jrandiny/yoga-slim7-ubuntu/issues/10)).



### Audio
If you see only a `Dummy Output` device in your audio-devices list, it is caused by a regression on ALSA https://bugs.launchpad.net/ubuntu/+source/alsa-lib/+bug/1901922

Fix is on the way

In the meantime there are two temporary solution

1. Updating `alsa-ucm-conf`, `libasound2`, and `libasound2-data` to the latest version from the proposed repository (https://wiki.ubuntu.com/Testing/EnableProposed)
2. Adding `options snd-hda-intel index=1` to `/etc/modprobe.d/alsa-base.conf` and reboot

**Warning** The second workaround will break HDMI audio out, please use the first solution if possible or wait for the official fix from Ubuntu 

## Thanks
- @SteveImmanuel for the information regarding microphone on kernel 5.7
- @nopmop for the audio workaround
- https://www.reddit.com/r/linuxhardware/comments/i28nm5/ideapad_14are05_s3_sleep_fix/ 
- https://wiki.archlinux.org/index.php/Lenovo_ThinkPad_X1_Yoga_(Gen_3)#Enabling_S3_(before_BIOS_version_1.33)
