# NixOS on the HoneyComb / ClearFog LX2k

This repo contains minimal viable Nix expressions for NixOS installer images and UEFI
firmwares for the following machines:

* [HoneyComb LX2K][lx2k-store]

# Prerequisites

Before starting, you should ensure you have:

* Any additional required hardware is properly installed.
  The default memory speed is 3200mhz. If your memory is slower, you will need to specify a different speed.
* A Linux machine with Nix installed.
* FTDI drivers configured according to https://developer.solid-run.com/knowledge-base/serial-connection/.
  (NixOS default kernel users are already covered!)
* A microUSB to USB cable. ([Ex][parts-microusb-to-usb-cable-ex])
* A microSD card. ([Ex][parts-microsd-card-ex])
* A USB stick, 4+ GB prefered. ([Ex][parts-usb-stick-ex])

# Usage

These machines require both a UEFI image (typically loaded on the microSD card) as well as an
installer image (typically loaded on a USB stick).

To get started, build the required images:

```bash
nix-build release.nix -A isoImage -o isoImage
nix-build release.nix -A lx2k.uefi -o uefi.img
```

Use `lsblk` to take a look at the block devices within `/dev`:

```bash
$ lsblk -o NAME,SIZE,MODEL,SERIAL,STATE
NAME          SIZE MODEL                     SERIAL             STATE
sda         465.8G Samsung_SSD_860_EVO_500GB S3Z1NB0K431062D    running
└─sda1      465.8G                                              
sdb           7.5G USB_Flash_Drive           AASB9S2CIA58DPXH   running # <-- USB Stick
sdc         238.9G UHSII_uSD_Reader          201912001648       running # <-- microSD card
nvme1n1     349.3G INTEL SSDPED1K375GA       PHKS750500HT375AGN live
├─nvme1n1p1   512M                                              
└─nvme1n1p2 348.8G                                                                                          
```

Then, copy the built images to those devices:

```bash
$ sudo cp -vi uefi.img /dev/sdc
cp: overwrite '/dev/sdc'? y
'uefi.img' -> '/dev/sdc'
$ sudo cp -vi isoImage/iso/*.iso /dev/sdb
cp: overwrite '/dev/sdb'? y
'isoImage/iso/nixos-21.03pre263901.68398d2dd50-aarch64-linux.iso' -> '/dev/sdb'
```

# Booting

Plug the microUSB side of the microUSB cable into the port which registers `dmesg` as `FTDI`, not the `STM32`
(see step 2). Plug the other side into the host machine with FTDI drivers.

```bash
$ dmesg | grep FTDI
# ...
[...] usb 3-3: FTDI USB Serial Device converter now attached to ttyUSB0
```

Modify the `kermit-config`'s `set line /dev/ttyUSB${NUMBER}` option to represent the `ttyUSB${NUMBER}` shown in `dmesg` above.

```bash
$ sudo nix-shell -p --run "kermit kermit-config"
Connecting to /dev/ttyUSB0, speed 115200
 Escape character: Ctrl-\ (ASCII 28, FS): enabled
Type the escape character followed by C to get back,
or followed by ? to see other options.
----------------------------------------------------

```

Next the fun part! **Hit the power button 🚀!**

With any luck, something similar to the following will show:

```bash
$ sudo nix-shell -p --run "kermit kermit-config"
Connecting to /dev/ttyUSB0, speed 115200
 Escape character: Ctrl-\ (ASCII 28, FS): enabled
Type the escape character followed by C to get back,
or followed by ? to see other options.
----------------------------------------------------
NOTICE:  BL2: v1.5(release):
NOTICE:  BL2: Built : 00:00:00, Jan  1 1980
NOTICE:  UDIMM KHX3200C20S4/32GX 
NOTICE:  DDR4 UDIMM with 2-rank 64-bit bus (x8)

NOTICE:  64 GB DDR4, 64-bit, CL=20, ECC off, 256B, CS0+CS1
NOTICE:  BL2: Booting BL31
NOTICE:  BL31: v1.5(release):
NOTICE:  BL31: Built : 00:00:00, Jan  1 1980
NOTICE:  Welcome to LX2160 BL31 Phase

UEFI firmware built at 00:00:00 on Jan  1 1980. version:

SOC: LX2160ACE Rev2.0 (0x87360020)
UEFI firmware (version  built at 00:00:00 on Jan  1 1980)
```

After, a UEFI boot menu will be presented. From here, proceed like a normal NixOS install.

# Notes

* `make menuconfig`

  Requires some dependencies:

  ```bash
  nix-shell -E 'with (import <nixpkgs> {}); stdenv.mkDerivation { name = "fake"; nativeBuildInputs = [ ncurses pkgconfig bison flex ]; }'

  export NIX_CFLAGS_LINK=$(pkg-config --libs ncurses)
  ```
# Future work

* **TODO:** Improve self-hosting
  + Determine which of =boot.kernelParams= are important and move them to
    a module that can be included by installed systems.
* Why is 2gb of memory allocated to huge pages?
* Does the PCI-E slot work?
* Discuss installation steps onto eMCC
* Create joined sdcard/isoimage for flashing to sdcards.
  + Add entry in UEFI for recovery with this image.

# LICENSE

This repo is a port of [Solidrun/lx2160a_build][github-solidRun-lx2160a_build], and has a copy of
the patches from that repository, it is subject to the same conditions. Anything original to this
repository is available under the same conditions as nixpkgs for ease of upstreaming.

[github-solidRun-lx2160a_build]: https://github.com/SolidRun/lx2160a_build
[lx2k-store]: https://shop.solid-run.com/product/SRLX216S00D00GE064H08CH/
[parts-microusb-to-usb-cable-ex]: https://www.memoryexpress.com/Products/MX30019
[parts-microsd-card-ex]: https://shop.solid-run.com/product/MSD016B/
[parts-usb-stick-ex]: https://www.memoryexpress.com/Products/MX64592
