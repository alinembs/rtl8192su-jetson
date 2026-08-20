# RTL8192SU — Linux 5.15 / NVIDIA Jetson Compatibility

Compatibility work for the legacy Realtek RTL8188SU / RTL8191SU /
RTL8192SU wireless driver on modern Linux kernels, with a focus on
NVIDIA Jetson platforms running Linux 5.15.

This repository is based on the original `rtl8192su` driver developed
by the Linux wireless community and contributors, including code from
Realtek Corporation.

The main purpose of this repository is to document and preserve the
kernel-compatibility modifications required to build and operate the
driver on an NVIDIA Jetson Orin NX running Linux 5.15.148-tegra.

---

## Table of Contents

- [Overview](#overview)
- [Hardware Tested](#hardware-tested)
- [Software Environment](#software-environment)
- [Original Project](#original-project)
- [Why This Fork Exists](#why-this-fork-exists)
- [Compatibility Work](#compatibility-work)
- [Build Result](#build-result)
- [Firmware](#firmware)
- [Installation](#installation)
- [Loading the Driver](#loading-the-driver)
- [Verifying the Adapter](#verifying-the-adapter)
- [Connecting to Wi-Fi](#connecting-to-wi-fi)
- [Automatic Module Loading](#automatic-module-loading)
- [Troubleshooting](#troubleshooting)
- [Known Limitations](#known-limitations)
- [Original Project Limitations](#original-project-limitations)
- [Changelog](#changelog)
- [Credits](#credits)
- [License and Copyright](#license-and-copyright)
- [Disclaimer](#disclaimer)

---

## Overview

The RTL8192SU family is based on an older Realtek USB wireless
chipset. Modern Linux kernels have changed several kernel APIs used by
the original driver, which can prevent the driver from compiling or
loading on newer systems.

This repository documents the compatibility work required to build the
driver against:

- Linux kernel 5.15
- NVIDIA Jetson Linux
- ARM64
- NVIDIA Jetson Orin NX

The driver was successfully compiled as kernel modules and loaded on
the target system. A Realtek RTL8188SU USB adapter was detected by
`cfg80211`/`mac80211` and successfully exposed a wireless network
interface.

---

## Hardware Tested

### Target platform

**NVIDIA Jetson Orin NX**

Architecture:

```text
ARM64 / AArch64
USB Wi-Fi adapter
```

Realtek RTL8188SU USB wireless adapter.

The adapter was identified by USB as:
```text
Realtek RTL8188SU
```
After loading the driver, the wireless interface was exposed as:
```text
wlxf8d1115eee8f
```
The interface name may be different on other adapters because Linux
can derive the interface name from the device MAC address.

## Software Environment

The compatibility work was tested with:
```
Operating System: Ubuntu 22.04
Kernel: 5.15.148-tegra
Architecture: ARM64
Compiler: GCC 11.4
Platform: NVIDIA Jetson Orin NX
```
Kernel build directory:
```
/lib/modules/5.15.148-tegra/build
```
The kernel source/headers used for the module build were:
```
/usr/src/linux-headers-5.15.148-tegra-ubuntu22.04_aarch64/
```
### Original Project

This repository is based on:

chunkeey/rtl8192su

## Original repository:

https://github.com/chunkeey/rtl8192su

The original project is a work-in-progress implementation of the
RTL8192SU driver and contains code from the Linux wireless subsystem
and Realtek Corporation.

The original source tree contains copyright and GPLv2 notices in
individual source files.

For example, the source files identify:
```
Copyright(c) 2009-2012 Realtek Corporation.
```
and refer to the GNU General Public License version 2.

The original repository should be considered the authoritative source
for the original driver history.

## Why This Fork Exists

The original driver targets an older Linux kernel API.

When building it against Linux 5.15.148-tegra, compilation initially
failed because several APIs used by the driver had changed or had
been removed.

The main goal of this work was therefore not to redesign the driver,
but to provide the minimum compatibility changes necessary to build
and test the existing RTL8192SU implementation on the Jetson platform.

This repository can also serve as a technical reference for developers
attempting to use legacy Realtek USB Wi-Fi hardware with newer Linux
kernels.

## Compatibility Work
1. Legacy kernel time API

The original source contained kernel time-related definitions that
were incompatible with the target Linux kernel.

The affected code was adapted to use the time type expected by the
5.15 kernel headers.

This was required before the driver could progress through the build.

2. mac80211 rate-control API

The original driver uses a legacy rate-control implementation.

The first build against Linux 5.15 produced:
```
error: implicit declaration of function
'rate_control_send_low'
```
The function was not available in the target kernel's mac80211 API.

The affected code was:
```
if (rate_control_send_low(sta, priv_sta, txrc))
    return;
```
The compatibility modification removes the dependency on this
obsolete API so that the driver's rate-control implementation can be
compiled against the available mac80211 interface.

3. struct rate_control_ops API change

The original driver defined:
```
static void *rtl_rate_alloc(struct ieee80211_hw *hw,
                            struct dentry *debugfsdir)
```
However, the Linux 5.15 kernel expects a different function signature
for the .alloc member of:
```
struct rate_control_ops
```
The original code therefore produced:
```
error: initialization of ‘void * (*)(struct ieee80211_hw *)’
from incompatible pointer type
‘void * (*)(struct ieee80211_hw *, struct dentry *)’
```
The function was adapted to match the API expected by the target
kernel.

4. Resulting rate-control registration

After the compatibility changes, the driver successfully registered
its rate-control algorithm:
```
ieee80211 phy0: Selected rate control algorithm 'rtl_rc'
```
This confirmed that the driver was able to integrate with the
mac80211 rate-control subsystem of the target kernel.

## Build Result

After the compatibility modifications, the complete module build
finished successfully.

The following kernel modules were generated:
```
rtlwifi/rtlwifi.ko
rtlwifi/rtl_usb.ko
rtlwifi/rtl_pci.ko
rtlwifi/rtl8192s/rtl8192s-common.ko
rtlwifi/rtl8192su/rtl8192su.ko
```
The build also generated the corresponding module metadata files.

Successful build output included:
```
LD [M] rtlwifi/rtl_usb.ko
LD [M] rtlwifi/rtlwifi.ko
LD [M] rtlwifi/rtl8192s/rtl8192s-common.ko
LD [M] rtlwifi/rtl_pci.ko
LD [M] rtlwifi/rtl8192su/rtl8192su.ko
```
## Firmware

The RTL8192SU driver requires firmware.

The driver source specifies:
```
#define RTL8192SU_FW_NAME "RTL8192SU/rtl8192sfw.bin"
```
Therefore the firmware must be available at:
```
/lib/firmware/RTL8192SU/rtl8192sfw.bin
```
The firmware used during testing was:
```
rtl8192sfw.bin
```
### Firmware License

The firmware is not the same licensing component as the driver
source.

The firmware distribution contains:
```
firmwares/LICENCE.rtlwifi_firmware.txt
```
The license identifies:
```
Copyright (c) 2010, Realtek Semiconductor Corporation
All rights reserved.
```
The firmware license permits redistribution in binary form without
modification subject to its stated conditions.

The firmware must therefore not be treated as GPL-licensed source code.

Read the complete firmware license before redistributing the firmware:

```
firmwares/LICENCE.rtlwifi_firmware.txt
```
## Installation
1. Install the required build dependencies

On Ubuntu 22.04:
```bash
sudo apt update
sudo apt install build-essential linux-headers-$(uname -r) git
```
On NVIDIA Jetson systems, make sure that the kernel headers correspond
to the exact running kernel.

Check:
```bash
uname -r
```
Expected:
```text
5.15.148-tegra
```
Verify the build directory:
```bash
ls -ld /lib/modules/$(uname -r)/build
```
2. Build the driver

From the repository root:
```bash
make clean
make -j$(nproc)
```
For the Jetson configuration used during testing, the build invokes
the kernel module build system with the RTL8192SU configuration enabled.

The resulting modules should be present under:

```text
rtlwifi/
```
Verify:
```bash
find rtlwifi -name "*.ko" -print
```
### Installing the Firmware

Create the firmware directory:
```bash
sudo mkdir -p /lib/firmware/RTL8192SU
```
Copy the firmware:
```bash
sudo cp firmwares/rtl8192sfw.bin \
    /lib/firmware/RTL8192SU/
```
Verify:
```bash
ls -lh /lib/firmware/RTL8192SU/rtl8192sfw.bin
```
Expected output should show the firmware file.

## Loading the Driver

The driver depends on several modules.

Load them in the following order:
```bash
sudo modprobe mac80211


sudo insmod rtlwifi/rtlwifi.ko
sudo insmod rtlwifi/rtl_usb.ko
sudo insmod rtlwifi/rtl8192s/rtl8192s-common.ko
sudo insmod rtlwifi/rtl8192su/rtl8192su.ko
```
If the driver loads correctly, the kernel log should contain messages
similar to:
```bash
rtl8192s_common: Chip version 0x3
rtl_usb: rx_max_size 9100, rx_urb_num 8, in_ep 3
rtl8192su: Driver for Realtek RTL8192SU/RTL8191SU
Loading firmware RTL8192SU/rtl8192sfw.bin
ieee80211 phy0: Selected rate control algorithm 'rtl_rc'
usbcore: registered new interface driver rtl8192su
```
The exact messages may vary depending on the kernel and hardware.

## Verifying the Adapter

Check wireless interfaces:
```bash
iw dev
```
A successfully initialized adapter should produce output similar to:
```text
phy#0
    Interface wlxf8d1115eee8f
        ifindex 13
        wdev 0x1
        addr f8:d1:11:5e:ee:8f
        type managed
        txpower 20.00 dBm
```
The interface name will normally differ between devices.

You can also check:
```bash
ip link
```
The wireless interface should appear as a network device.

### Checking Kernel Messages

Use:
```bash
dmesg | grep -Ei "rtl8192|rtlwifi|firmware|wlan|wlx"
```
Or:
```bash
sudo dmesg | tail -100
```
A successful initialization should include firmware loading and
registration of the RTL8192SU USB driver.

## Connecting to Wi-Fi

Once the interface is available, it can be managed using the normal
Linux networking stack.

First check:
```bash
iw dev
```
Then scan:
```bash
sudo iw dev wlxf8d1115eee8f scan | less
```
Replace wlxf8d1115eee8f with the interface name reported by your
system.

If NetworkManager is installed, the interface can also be managed
using:
```bash
nmcli device status
```
and:
```bash
nmcli device wifi list
```
Connect using:
```bash
nmcli device wifi connect "SSID" password "PASSWORD"
```
Replace SSID and PASSWORD with the appropriate network
credentials.

## Automatic Module Loading

The current development procedure loads the modules manually using
insmod.

For a permanent installation, the modules should be installed into
the kernel module tree and registered with the module dependency
system.

For example, after installing the modules:
```bash
sudo depmod -a
```
The exact installation procedure may vary depending on the NVIDIA
Jetson Linux distribution and kernel packaging.

Before configuring automatic loading, confirm that the manually loaded
driver works correctly.

## Troubleshooting
### Firmware not found

If the kernel reports a firmware loading error, check:
```bash
ls -l /lib/firmware/RTL8192SU/
```
The required file is:
```bash
rtl8192sfw.bin
```
The complete path must be:
```bash
/lib/firmware/RTL8192SU/rtl8192sfw.bin
```
Check the kernel log:
```bash
dmesg | grep -i firmware
```
### rate_control_send_low compilation error

If you see:
```bash
error: implicit declaration of function
'rate_control_send_low'
```
you are probably compiling an unmodified version of the legacy driver
against a kernel whose mac80211 API no longer provides that function.

Check the compatibility modifications in:
```bash
rtlwifi/rc.c
```
### rtl_rate_alloc incompatible pointer type

If you see:
```bash
error: initialization of ‘void * (*)(struct ieee80211_hw *)’
from incompatible pointer type
```
check the definition of:
```bash
rtl_rate_alloc()
```
and compare it with the rate_control_ops definition in the kernel:
```bash
include/net/mac80211.h
```
The function signature must match the kernel API.

### Compiler warning

The Jetson build may report:
```bash
warning: the compiler differs from the one used to build the kernel
```
For example:
```bash
The kernel was built by:
gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0


You are using:
gcc (Ubuntu 11.4.0-1ubuntu1~22.04.3) 11.4.0
```
This particular difference is a package revision difference while the
compiler version remains GCC 11.4.0.

It did not prevent the driver from building during testing.

**Module signature warning**

When manually loading an externally built kernel module, the kernel
may report:
```bash
rtlwifi: module verification failed:
signature and/or required key missing - tainting kernel
```
This indicates that the module is not signed with a key trusted by the
kernel.

It does not necessarily indicate a functional problem with the
driver.

For production deployments, investigate the kernel's module-signing
requirements before relying on unsigned modules.

**Adapter is detected but no wireless interface appears**

Check:
```bash
lsusb
```
Then:
```bash
lsmod | grep -E "rtl8192su|rtl8192s|rtl_usb|rtlwifi"
```
And:
```bash
dmesg | grep -Ei "rtl8192|rtlwifi|usb|firmware"
```
Also verify that the USB device remains connected:
```bash
lsusb
```
USB stability is particularly important on embedded platforms.

## Known Limitations

This compatibility work does not imply that every feature of the
original RTL8192SU driver has been validated on Linux 5.15.

The current testing primarily establishes:

- Driver compilation.
- Module loading.
- Firmware loading.
- USB device initialization.
- mac80211 registration.
- Wireless interface creation.
- Wi-Fi connectivity.

Additional functionality requires further testing.

## Original Project Limitations

The original repository documents several limitations, including:

- No AP/P2P mode.
- No accurate TX feedback due to firmware/host protocol limitations.
- 802.11n functionality requires additional validation.
- Signal/quality statistics may be inaccurate.
- RX statistics require additional work.
- Locking/RCU behavior requires further validation.

See the original project's TODO file for additional information.

## Changelog
**2026-08-19 — Linux 5.15 / Jetson Compatibility**
**Added**
- Compatibility work targeting Linux 5.15.148-tegra.
- NVIDIA Jetson Orin NX validation.
- RTL8188SU USB adapter validation.
- RTL8192SU firmware loading.
- Documentation of the build and installation process.
**Fixed**
- Legacy kernel time API compatibility issue.
- Removed dependency on the obsolete rate_control_send_low()
- mac80211 function.
- Updated the RTL rate-control allocation function to match the
- Linux 5.15 struct rate_control_ops API.
**Tested**
```bash
Platform: NVIDIA Jetson Orin NX
Architecture: ARM64
Kernel: 5.15.148-tegra
OS: Ubuntu 22.04
Compiler: GCC 11.4
USB adapter: Realtek RTL8188SU
Firmware: rtl8192sfw.bin
```
**Result**

The driver was successfully:

1. Compiled.
2. Loaded into the kernel.
3. Initialized the RTL8192SU chipset.
4. Loaded the required firmware.
5. Registered with mac80211.
6. Created a wireless network interface.
7. Connected to a Wi-Fi network.
## Credits
**Original Driver**

This project is based on the original:

**rtl8192su**

Repository:

https://github.com/chunkeey/rtl8192su

The original source contains contributions from the Linux wireless
community and Realtek Corporation.

Copyright notices present in the source files must be preserved.

### Compatibility Work

Linux 5.15 / NVIDIA Jetson compatibility modifications and hardware
validation:

**Aline Mariana — 2026**

This work focuses on adapting the existing driver to the APIs provided
by the Linux 5.15 kernel used by the tested NVIDIA Jetson platform.

### AI-Assisted Development

AI-assisted debugging, API compatibility analysis and troubleshooting
were performed using:

**OpenAI ChatGPT (GPT-5.6 Luna)**

The final modifications were reviewed, applied and tested by the
maintainer.

AI assistance does not replace the original copyright or authorship of
the driver code.

## License and Copyright

The original source files contain copyright and licensing notices
associated with their respective authors and copyright holders.

Several source files explicitly reference the GNU General Public
License version 2 (GPLv2).

Copyright notices from the original source must remain intact.

The repository should therefore be treated as a derivative work of the
original driver, with the original licensing terms applying to the
corresponding source code.

The firmware is separately licensed by Realtek Semiconductor
Corporation and is not covered by the driver source license.

See:
```bash
firmwares/LICENCE.rtlwifi_firmware.txt
```
before redistributing firmware.

## Disclaimer

This project is provided for research, development, compatibility
testing and educational purposes.

The RTL8192SU driver is an old/WIP driver and its behavior on modern
kernels and embedded platforms may vary.

Successful operation on the tested NVIDIA Jetson Orin NX configuration
does not guarantee compatibility with other Jetson models, kernel
versions, USB adapters or firmware revisions.

Use the driver at your own risk.

## Contributing

Contributions are welcome.

If you test this driver on another platform or kernel version, please
include:

- Hardware model.
- USB chipset.
- Kernel version.
- Linux distribution.
- Architecture.
- Firmware version/file.
- Build output.
- dmesg output.
- iw dev output.
- Description of the observed behavior.

When reporting a problem, please avoid posting passwords, Wi-Fi
credentials, private keys or other sensitive information.

## Reproducibility

The goal of this repository is to make the compatibility work
reproducible.

A minimal test report should contain:
```bash
uname -a
```
```bash
lsusb
```
```bash
iw dev
```
```bash
lsmod
```
and:
```bash
dmesg | grep -Ei "rtl8192|rtlwifi|firmware"
```
These commands provide enough information to identify many common
compatibility problems.

### Status

Working on tested configuration

```bash
✓ Driver compilation
✓ Kernel module generation
✓ Firmware loading
✓ RTL8192SU chipset initialization
✓ mac80211 registration
✓ USB wireless interface creation
✓ Wi-Fi connectivity
```

⚠ Additional driver functionality requires further validation
### Project Goal

The long-term goal is to preserve a practical path for using legacy
Realtek USB wireless hardware on modern Linux-based embedded systems,
while documenting the kernel API changes required to keep the driver
buildable and testable.

If this project helps you bring an old RTL8192SU-family adapter back
to life on a modern Linux system, consider documenting your hardware,
kernel version and results so that the compatibility information can
benefit other developers.
