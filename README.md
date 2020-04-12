# Serial WiFi Bridge

2020-04-11 Sascha Schade

## Problem

**Concrete** An industrial inverter that can only be reconfigured over a
serial link was placed in a cabinet not easily accessible. The Windows
software (IndraWorks from Bosch Rexroth) can only access local serial ports.

**Abstract** Access a serial port wirelessly in a remote location from within
a Windows VM.

## Solution

The solution is to tunnel the serial protocol over WiFi to a virtual COM port
into the virtual machine that is running Windows and IndraWorks.

## Architecture

```
  +--------------+  MiniDIN 8 poles
  |  IndraDrive  |   3: TxD
  |   Inverter   |   4: GND
  +--------------+   5: RxD
         | RS232 Serial Link
  +--------------+
  | RS232 to USB |
  |    PL2303    |
  +--------------+
         | USB 2.0
  +--------------+
  |   OpenWrt    | TP-WR703N
  |   ser2net    | Port 8211
  +--------------+
         | Wifi company intern
+------------------+
| +--------------+ |
| |   com2tcp    | |
| +--------------+ |
|        | COM4    |
| +--------------+ |
| |   com2com    | |
| +--------------+ |
|        | COM3    |
| +--------------+ |
| |  IndraWorks  | |
| +--------------+ |
|    Windows VM    |
+------------------+
```

* On Windows, `com0com` is used to create a pair of connected virtual COM
  ports (`COM3` and `COM4`).
* Then, `com2tcp` connects one of them (`COM4`) to the remote COM port served
  by `ser2net`.
* The application then can connect to the other port of the connected pair
  (`COM3`).
* All traffic written into `COM3` on the Windows machine is tunneled to the
  USB serial adaptor connected to the inverter.
* All traffic which arrives at the USB to serial adaptor is tunneled back to
  `COM3` and can be read by the application.

Latency is small enough so that the application runs smoothly.

## Setup

The setup starts from scratch with a small TP-Link TL-WR703N v1. Any other
WiFi router with at least one USB 2.0 port and decent OpenWrt support can be
used.

### Compile OpenWrt

OpenWrt is compiled from source (master, 19.07, commit `r12903`, `c30220d458`
at time of writing) with a monolith firmware image. The image contains only
the necessary components, so that the 4 MiB flash and 32 MiB RAM are not a
limit. Noteworthy, IPv6 (sadly) and PPP are disabled, ser2net and USB serial
driver for PL2303 are enabled.

Install steps on macOS:

```
hdiutil create -size 20g -type SPARSE -fs "Case-sensitive HFS+" -volname OpenWrt OpenWrt.sparseimage
hdiutil attach OpenWrt.sparseimage
cd /Volumes/OpenWrt
git clone https://git.openwrt.org/openwrt/openwrt.git
cd openwrt
export PKG_CONFIG_PATH="/usr/local/opt/ncurses/lib/pkgconfig"
export CPPFLAGS="-I/usr/local/opt/ncurses/include"
export LDFLAGS="-L/usr/local/opt/ncurses/lib"
export PATH="/usr/local/opt/ncurses/bin:$PATH"
export PATH="/usr/local/opt/gnu-getopt/bin:$PATH"
./scripts/feeds install ser2net
./scripts/feeds update -a
make prereq
```

The `.config` file is part of this repository. Config options were:

* Target System: `Atheros AR7xxx/AR9xxx`
* Subtarget: `Devices with small flash`
* Target Profile: `TP-LINK TL-WR703N v1`
* Kernel modules: USB Support:
  - `kmod-usb-serial`
  - `kmod-usb-serial-pl2303`
* Network: enable `ser2net`

Compile the build system, the kernel and all packages. This may take some time.

```
make -j8
```

### Provide firmware image on TFTP server

Luckily, macOS has an integrated TFTP server. Enable it with

```
sudo launchctl load -F /System/Library/LaunchDaemons/tftp.plist
sudo launchctl start com.apple.tftpd
```

Copy the firmware image to the TFTP server directory:

```
sudo cp ./bin/targets/ar71xx/tiny/openwrt-ar71xx-tiny-tl-wr703n-v1-squashfs-factory.bin /private/tftpboot/firmware.bin
```

### Flash OpenWrt

Set the IP address of your wired Ethernet adaptor to `192.168.1.100`. That is
the default server IP of the U-Boot bootloader. A different IP can be set with
`setenv` within the bootloader. Connect the Ethernet to you computer (either
directly or with a switch) and the router.

Connect an USB to serial adaptor to the serial pins of WR703N. Settings are
115200 8N1. Open a terminal:

```
picocom -b 115200 --imap lfcrlf /dev/tty.usbserial-0001
```

Boot the device. Enter `tpl` quickly once the bootloader waits for a second.
You should have arrived at the U-Boot prompt `hornet>`.

Get the firmware image from the TFTP server, erase the flash, write the
firmware and boot into the new image:

```
tftpboot 0x81000000 firmware.bin
erase 0x9f020000 +0x3c0000
cp.b 0x81000000 0x9f020000 0x3c0000
bootm 9f020000
```

### Configure OpenWrt

Set a root password with `passwd`.

Configure WiFi, network and ser2net:

* `/etc/config/network`
* `/etc/config/wireless`
* `/etc/config/ser2net`

Default configuration is in the repository and remember to adjust SSID and
password in `wireless`. Do not get confused with the `ser2net` settings in
`/etc/ser2net.conf` and `/etc/config/ser2net.conf`

General configuration is:

* WiFi in Station Mode, connect to the company WiFi
* ser2net configured to listen on port `8211` in Telnet mode and serves
  `/dev/ttyUSB0`.

### Configure Windows (in VM)

Install [`com0com`](https://github.com/Raggles/com0com/releases) and configure
a bridge between `COM3` and `COM4`.

`com2tcp` must be compiled from source using Visual Studio.
[U-Blox](https://github.com/u-blox/com2tcp) has an updated version.
Unfortunatly, no CI is setup yet.

To discover the device when using DHCP, `nmap` can scan the network for the
specific open port:

```
nmap -Pn -p 8211 172.20.20.0/24
```
Alternatively, the MAC address can be searched with
```
arp -a
```


`com2tcp` runs from the command shell:

```
com2tcp.exe --telnet --ignore-dsr --baud 115200 \\.\COM4 ${IP} ${PORT}
```

## Discussion

The solution is using readily available components and is accessible over the
company WiFi.

Alternative solutions could have been using the router as an access point with
the disadvantage that then a connection to the Internet of the laptop is then
discontinued.

Other hardware could also do the job:

* ESP8266 or ESP32 module with a RS232 to TTL converter, directly attached to
  the inverter
* Bluetooth serial bridges (delivered in pairs)
* STM32 with Ethernet (once Ethernet is routed to the cabinet)
* A more industrial grade solution (without WiFi) could be realised with
  something like Phytec phyBOARD-Regor which has C-rail mounting, 24 VDC input
  and hardware RS232 on board. Unfortunately, no Ethernet is yet routed to the
  cabinet and the board uses the RS232 as the kernel console. Disabling the
  kernel console is quite inconvenient for debugging.
