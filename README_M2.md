# Building and Booting Fractal on an M2 Mac Mini

This file describes a from-scratch workflow for building Fractal for an Apple
M2 Mac mini and loading it through m1n1.

## Tested Target

The tested machine is:

```text
Model: Mac14,3
Target: J473
Chip-ID: 0x8112
CPU: M2 Blizzard
```

The relevant addresses read from the ADT are:

```text
UART: /arm-io/uart0 = 0x235200000
WDT:  /arm-io/wdt   = 0x23d2b0000
AIC:  /arm-io/aic   = 0x23b0c0000
```

`SOC=M2` currently uses the UART and WDT addresses above. The AIC address is
reported by m1n1 but is not currently consumed by Fractal's Apple board code.

## Host Requirements

Use a second Mac as the host machine. Connect the host Mac to the target M2 Mac
mini with a USB SuperSpeed cable connected to the target's DFU port.

Install the host-side tools:

```bash
brew install picocom
```

Install `macvdmtool` from the Asahi Linux project:

```text
https://github.com/AsahiLinux/macvdmtool
```

You also need Python packages required by m1n1's `proxyclient`, including
`pyserial` and `construct`.

## Get the Source

Clone this repository:

```bash
git clone https://github.com/JinningL/fractal.git
cd fractal
```

## Build the Fractal Toolchain

Build only the Apple Silicon toolchain:

```bash
cd toolchain
./mk_toolchain.sh apple
cd ..
```

This creates tools under:

```text
toolchain/out/bin
```

Add them to `PATH`:

```bash
export PATH="$PWD/toolchain/out/bin:$PATH"
```

You should now have tools such as:

```text
aarch64-fractal-elf-gcc
aarch64-fractal-elf-g++
aarch64-fractal-elf-objcopy
genext2fs
```

## Create the Initial Filesystem

Fractal's Apple image appends an aarch64 ext2 initrd at:

```text
filesys/disk.aarch64.ext2
```

Create a minimal filesystem image:

```bash
cd filesys
make
cd ..
```

This creates an empty `filesys/fs_aarch64/bin` directory and packages it into
`filesys/disk.aarch64.ext2`.

For a useful boot, place your Fractal aarch64 userspace program in:

```text
filesys/fs_aarch64/bin/
```

The existing experiment workflow expects a program named:

```text
filesys/fs_aarch64/bin/utest
```

After adding or replacing files in `filesys/fs_aarch64`, rebuild the initrd:

```bash
cd filesys
make clean
make
cd ..
```

## Build the M2 Kernel Image

Build Fractal for the M2 Mac mini:

```bash
make -j BOARD=APPLE SOC=M2 arm
```

The final image is:

```text
bin/fractal.release.apple.arm.img
```

That file is the Apple kernel payload plus the appended aarch64 ext2 initrd.

## Get m1n1

This workflow was tested with m1n1 reporting this build tag on the serial
console:

```text
m1n1 v1.5.0-101-g64762a7
```

Use that m1n1 revision, or a newer upstream m1n1 revision with support for the
M2 Mac mini (`Mac14,3` / `J473` / `T8112`). The local copied m1n1 tree used
during testing did not include its own `.git` directory, so the serial build
tag above is the version identifier to match.

Clone m1n1 next to or inside this repository:

```bash
git clone https://github.com/AsahiLinux/m1n1.git
```

For a reproducible checkout, use the tested commit if it is available in the
upstream repository:

```bash
cd m1n1
git checkout 64762a7
cd ..
```

The examples below assume m1n1 is inside the Fractal checkout:

```text
fractal/m1n1
```

If it is elsewhere, adjust the paths accordingly.

## Put the Target Mac in Serial Mode

From the host Mac:

```bash
sudo macvdmtool reboot serial
```

Wait for the m1n1 USB serial devices to appear:

```bash
ls /dev/tty.usbmodem*
```

You may see two devices, for example:

```text
/dev/tty.usbmodemYKN407WGX71
/dev/tty.usbmodemYKN407WGX73
```

Use the one that responds to m1n1. If one times out, try the other.

## Chainload Fractal

Set the m1n1 device:

```bash
export M1N1DEVICE=/dev/tty.usbmodemYKN407WGX71
```

Load Fractal:

```bash
cd m1n1
python3 proxyclient/tools/chainload.py -n -r -t p -E 0 \
  ../bin/fractal.release.apple.arm.img
```

Successful output includes:

```text
Fetching ADT ...
Loading kernel image ...
Entry point: 0x...
Starting CPU 1 ... Started.
...
Proxy is alive again
```

`chainload.py` returning to the shell is normal. It only loads the image; it is
not the Fractal console.

## Read Fractal Output

Use the debug console:

```bash
picocom -q --omap crlf --imap lfcrlf -b 115200 /dev/cu.debug-console
```

Or:

```bash
screen /dev/cu.debug-console 115200
```

To exit `screen`, press `Ctrl-A`, then `K`, then `y`.

## Local Convenience Files

During local testing it can be useful to keep:

```text
m1n1/
toolchain/out/
filesys/disk.aarch64.ext2
bin/
```

These are local tools or generated artifacts and should not be committed to the
Fractal source repository.
