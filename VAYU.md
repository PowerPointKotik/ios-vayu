# vayu (Poco X3 Pro, SM8150) bring-up

Base: `dipa-studio/ios-sweet` @ `425b251`.

This is a **development** port: it uses the proven `SWEET` kernel and the
sweet driver-facing ADT names, but the vayu physical memory window,
framebuffer size and stock vayu DTB.  Only the HFS+ iOS root in `userdata`
was flashed (by the user); boot images are tested with `fastboot boot`.

## Current artifacts

Older vayu images (vayu4-vayu15 and vayu17-vayu19, with their payloads) are archived on the
external drive `ios-vayu` at `/Volumes/ios-vayu/ios-sweet/shim/out/` (moved 2026-10-23 to free
the internal SSD; `SHA256SUMS` and `README.txt` live there).  Only `vayu16` (first bootable
touch build) and `vayu20` (current) stay in `shim/out/`.

| File | Purpose | SHA-256 |
| --- | --- | --- |
| `shim/out/shim-boot-vayu-diag.img` | `fastboot boot` image | `29f9faf4d80ee19d865dcaa8145aa485005cd732c296d9d43db67f0808e10a39` |
| `shim/out/payload-vayu-diag.bin` | XNU payload inside the image | `3cdce1419c6ceb941a81755e3e603509ea284dd2a93a760dc9d61b475d841b92` |
| `shim/out/vayu.adt` | vayu ADT (now with the real D22AP `/product`) | `23e3b7e01f45a6e10b177e5a1f396de4d596a23102fc13c7a4d2a51ede4b4444` |
| `xnu/out-0057/kernel.development.sweet` | CI kernel, 2026-10-18 19:25 (latest SWEET; `out-0057` is a historical local label) | `d9cb893bd2baf841e94cd2001f0ec70c0fea42f045f87f14ae807e4ebc50e1e5` |

The kernel came from the public CI artifact of run `39158169212`
(`nightly.link` mirror of the `xnu-kernel` artifact).  Despite the local
`out-0057` directory name, the extracted `SHA256SUMS` and the artifact
timestamp (2026-10-18 19:25) show the latest SWEET build: it contains
`SweetHID`, `SweetNET`, `sweetusb`, `sweetfb_hz` and the other changes up
to 0081.  It is the same `SWEET` build line that reaches the iOS home
screen on sweet.

## What is different for vayu

- `/chosen dram-base = 0xc0000000`, `dram-size = 0xbff00000`
  (3 GiB bank, panic log carved from the top at `0x17ff00000`). This bank is
  clear of every vayu reserved-memory carve-out (the highest low-bank one is
  `ramoops@0xb0000000`) and stays below 4 GiB for UFS DMA.
- `sweet-fbcon` covers `0x9c000000 + 0x2400000`, the vayu
  `cont_splash_region`.
- Root compatible gains `xiaomi,vayu` and `qcom,sm8150` **after**
  `xiaomi,sweet`; the SWEET platform expert still matches.
- CPU nodes keep `qcom,kryo470`, because `SweetCPU.cpp` matches that string
  exactly. vayu's Kryo 485 has the same MPIDR `Aff1:Aff0` layout
  (`0x000..0x700`), and the kernel reads the topology from the ADT.
- The shim build is portable (clang/lld or the usual
  `aarch64-linux-gnu-*`), takes `DTB=...`, and uses
  `tools/mkbootimg_v2.py` when AOSP `mkbootimg` is absent.
- `shim/loader.c` falls back to the vayu boot-image/stock-initrd addresses
  if the ABL leaves `/chosen/linux,initrd-start` pointing at a stale
  ramdisk. The first choice is still the DTB value, so sweet is unchanged.

## GUI / SpringBoard (framebuffer, no GPU)

This is not hypothetical.  The sweet base reached the iOS 16.7 home screen
on 2026-10-18 (kernel 0069 + root image v14) using the ABL splash
framebuffer, not an Adreno GPU.  The path is:

```text
SpringBoard/backboardd -> CoreAnimation (CPU) -> IOCoreSurfaceRoot /
IOSurface -> AppleCLCD (SweetFramebuffer) -> cont_splash_region
```

The vayu kernel here is relinked from the 2026-10-18 19:25 CI artifact
(the latest SWEET build line, through 0081) and contains `SweetHID`,
`SweetNET` and the IOSurface/IOMobileFramebuffer code.  vayu's
`cont_splash_region@0x9c000000` (size `0x2400000`) is exactly the
`FB`/`VAYU_FB_SIZE` in `tools/mkadt.py`, and `disp0` is 1125x2436; the
same display path applies.  The missing piece for a home screen is the
iOS HFS+ root image, not a GPU driver.  vayu touch/keys are the other
hardware dependency.  Wi-Fi, Bluetooth and a real GPU are not needed for
the minimal UI (they are separate, much longer projects).

The boot image `shim-boot-vayu-ufs-vayu-gui2.img` uses `sweetfb_noconsole=1`,
`sweetfb_hz=30` and `cpus=6`; its debug-console twin
`shim-boot-vayu-ufs-vayu2.img` uses `cpus=1` and keeps the kernel text so
the first boot can be diagnosed.  Both carry
`unentitled_ios_sim_launch=1 ignition_level=1 ufsrw=1 rootrw=1
launchd_unsecure_cache=1` (the working sweet boot arguments).

## Boot it (temporary only)

Put the phone in fastboot, then:

```sh
fastboot boot /Users/ppkotik/Documents/unixnu-vayu/ios-sweet/shim/out/shim-boot-vayu-diag.img
```

**Do not run `fastboot flash`.** The image only runs from RAM; it does not
write Android, boot, recovery, modem or persist.

The boot arguments are:

```
debug=0xc serial=3 cpus=1 keepsyms=1 -enable_kprintf_spam -noprogress
rd=ufsroot0 wdt=-1 sweetkeys=0 sweettouch=0 sweetusb=0 sweetbattery=0
```

`rd=ufsroot0` is read-only (`ufsrw` is not set). It is expected to find the
Android `userdata` partition and then reject its encrypted HFS signature.
That is the successful UFS/console milestone; it does **not** touch the
partition.

## What to look for

1. Shim screen (red, then lines):
   - `SHIM sweet milestone 4: xnu loader`
   - `INITRD lo=... hi=...`
   - `PHYS base=c0000000 size=bff00000`
   - `SEGS ...`, `BOOTARG ...`, `JUMP ...`
2. Kernel screen (after `XNU jumping`):
   - `SWEET` platform/CPU/GIC/timer lines
   - `SWEET FB: AppleCLCD 1080x2400 ... at 0x9c000000`
   - `SWEET UFS: "userdata" on LUN 0: ...`
   - an HFS signature/mount error or a mountroot retry.
3. If it stops earlier, photograph the last screen; the shim prints its
   stage and the kernel panics print `panic(...)` plus a backtrace.

The first boot is deliberately `cpus=1` and has touch/keys/USB/battery
drivers disabled, so a missing vayu PMIC/touch driver cannot hang the
kernel.

## Minimal userspace test (`rd=md0`, no Android writes)

Once the kernel reaches the root-device search, the next milestone is PID 1.
This image uses SWEET's `mockfs`: the RAM disk is a static Mach-O and is
served as `/sbin/launchd`, so no HFS+ image or Android partition is needed.

```sh
fastboot boot /Users/ppkotik/Documents/unixnu-vayu/ios-sweet/shim/out/shim-boot-vayu-md0.img
```

SHA-256:
`a8ae43f47690bc092f0273e7344db286a37a291ac1812d5b65fd0710e41ba166`

Expected after the kernel lines:

```
Added memory device md0
BSD root: md0, major 1, minor 0
load_init_program: attempting to load /sbin/launchd
*** SWEET userspace: init running as PID 1 ***
SWEET init: still alive
```

`SWEET init: still alive` repeats every 5 seconds. This proves console,
kernel/userland transition, exec, ad-hoc code signing and devfs
`/dev/console` on vayu. It is still a pure RAM boot: `rd=md0` does not
mount or write `userdata`.

Build it with:

```sh
clang -target arm64-apple-ios16.0 -c tools/sweet-init.S -o shim/out/sweet-init.o
python3 tools/mkinit.py shim/out/sweet-init.o shim/out/sweet-init

python3 tools/mkpayload.py \
  xnu/out-0057/kernel.development.sweet shim/out/vayu.adt \
  shim/out/payload-vayu-md0.bin --ramdisk shim/out/sweet-init \
  --cmdline 'debug=0xc serial=3 cpus=1 keepsyms=1 -enable_kprintf_spam -noprogress rd=md0 wdt=-1 sweetkeys=0 sweettouch=0 sweetusb=0 sweetbattery=0'

CC='clang --target=aarch64-none-elf' LDFLAGS='-fuse-ld=lld' \
OBJCOPY=/usr/local/opt/llvm/bin/llvm-objcopy \
READELF=/usr/local/opt/llvm/bin/llvm-readelf \
NM=/usr/local/opt/llvm/bin/llvm-nm \
DTB=/Users/ppkotik/Documents/unixnu-vayu/uefi-sheng/Resources/DTBs/vayu.dtb \
OUT=out/shim-boot-vayu-md0.img \
./shim/build.sh ../shim/out/payload-vayu-md0.bin
```

## HFS+ root test (`rd=md0`, still RAM only)

The next step is the filesystem the real iOS root will use. This 2 MB
case-sensitive HFS+ image has the same static init at `/sbin/launchd`; the
kernel's Apple OSS HFS driver mounts it as `VayuRoot`.

```sh
fastboot boot /Users/ppkotik/Documents/unixnu-vayu/ios-sweet/shim/out/shim-boot-vayu-hfs.img
```

SHA-256:
`3c9406446d957ef0a2504ac7e33b47f88c207ad5427923e3f816a8cd2f039f48`

Expected:

```
Added memory device md0
BSD root: md0, major 1, minor 0
hfs: mounted VayuRoot on device b(1, 0)
load_init_program: attempting to load /sbin/launchd
*** SWEET userspace: init running as PID 1 ***
SWEET init: still alive
```

This is still a RAM disk: no UFS mount, no Android writes. It validates
HFS+ plus the init path that the full iOS root will need.

The root image can be recreated on macOS (the device name from
`hdiutil attach` is dynamic):

```sh
dd if=/dev/zero of=vayu-root-hfs.img bs=1m count=2
hdiutil attach -nomount -imagekey diskimage-class=CRawDiskImage vayu-root-hfs.img
newfs_hfs -s -v VayuRoot /dev/diskN
hdiutil attach -imagekey diskimage-class=CRawDiskImage \
  -mountpoint /tmp/vayu-root-mnt vayu-root-hfs.img
mkdir -p /tmp/vayu-root-mnt/sbin /tmp/vayu-root-mnt/dev
cp shim/out/sweet-init /tmp/vayu-root-mnt/sbin/launchd
chmod 755 /tmp/vayu-root-mnt/sbin/launchd
hdiutil detach /dev/diskN
```

## Adreno 640 (GPU) stage 1, 2026-10-22

`vayu-port/ci/buildtree/xnu/bsd/dev/vayu_gpu.c` (called from
`bsd/dev/sweet_ufs.c` next to the input drivers) is the first, diagnostic
half of the GPU bring-up.  It never touches GX, the CP or a ringbuffer, so
it cannot disturb the framebuffer path.

What it does:

1. Maps GCC `0x100000`, GPUCC `0x2c90000`, the GMU `0x2c6a000` and the GPU
   core `0x2c00000`.  It reads the three GCC GPU branches and then enables
   them (`cfg-ahb` `0x71004`, `snoc-dvm-gfx` `0x71018`, `memnoc-gfx`
   `0x7100c`) even for the read-only probe: `cfg-ahb` gates every GPUCC
   register access and is `CLK_IS_CRITICAL` in Linux, and touching an
   unclocked GPUCC can hang the interconnect.  Then it prints the GDSCR
   power-domain state, the CX branch clocks, the GMU clock source and the
   CM3/HFI registers.
2. `vayugpu_power=1`: powers up the CX domain (GPUCC `CX_GDSCR` `0x106c`,
   `SW_COLLAPSE` bit 0) and enables `ahb`, `crc-ahb`, `cx-apb`,
   `cx-snoc-dvm`, `cxo-aon`, `cxo` and `cx-gmu`.
3. `vayugpu_fw=1`: reads `a640_gmu.bin` (block format), copies the ITCM/DTCM
   blocks through the CM3 TCM windows (`0x1b400`/`0x1c400`), fills the HFI
   queue table in contiguous memory and releases `CM3_SYSRESET`.  It then
   waits for the firmware's `0xBABEFACE` init result, starts HFI and asks
   the running firmware for its version.

Expected console output (probe + power, no firmware file yet):

```
VAYU GPU: Adreno 640 (chipid 0x6040001) probe, power=1 fw=1
VAYU GPU: [boot] gcc cfg-ahb=on(0x...) memnoc-gfx=... snoc-dvm-gfx=...
VAYU GPU: clk gcc-cfg-ahb on (0x...)
VAYU GPU: [boot] gpucc cx-gdscr=0x... gx-gdscr=0x... cx=... gx=...
VAYU GPU: [boot] clk ahb=... crc-ahb=... cx-apb=... cx-gmu=... ...
VAYU GPU: [boot] gmu-clock-src cmd=... cfg=... sysreset=... ...
VAYU GPU: CX domain on gdscr=0x...
VAYU GPU: clk cx-gmu on (0x...)
VAYU GPU: [powered] ...
VAYU GPU: a640_gmu.bin not found; put it in /usr/local/share/firmware/ or pass vayugpu_fwpath=
```

If the console stops right after the header line, the GPUCC read hung the bus:
that is exactly what the GCC clock enable in stage 2 is there to prevent, so
report where it stopped and use the next paragraph to get the clocks up
before any GPUCC access.

The GMU firmware is a vendor blob, not part of iOS.  Pull it from Android:

```sh
adb shell su -c 'cp /vendor/firmware/a640_gmu.bin /sdcard/'
adb pull /sdcard/a640_gmu.bin
```

Then either place it in the iOS root at
`/usr/local/share/firmware/a640_gmu.bin` or pass
`vayugpu_fwpath=/path/on/root/a640_gmu.bin`.  The driver also looks in
`/usr/share/firmware/`, `/private/var/vayu/` and `/vendor/firmware/`.

What is still missing, and why HFI fails right after `0xBABEFACE`: the kgsl
SMMU (`0x2ca0000`) has no context bank, so the GMU cannot read the HFI queue
table.  The following stages are the SMMU page tables, GX power-on
(`GX_GDSCR`, `gx-gmu`), the CP microcode (`a630_sqe.fw`), the ZAP shader
(`a640_zap`) and the ringbuffer; the iOS side then needs an `IOAccelerator`
service (the closed AGX ABI) before CoreAnimation can use the GPU.

Two images are provided: `shim-boot-vayu-md0-gpu.img` runs all three
stages, and `shim-boot-vayu-md0-gpuprobe.img` runs only the probe plus the
GCC branches, in case the CX power sequence needs to be ruled out.

Boot args: the probe is **off by default**; `vayugpu=1` enables it (plus
the three GCC GPU branches described above), `vayugpu_power=1` enables the
CX power/clock writes and `vayugpu_fw=1` enables the firmware/GMU stage
(and implies `vayugpu_power=1`).
Images built without `vayugpu=1` are unaffected by the driver.

Build the image:

```sh
cd /Users/ppkotik/Documents/unixnu-vayu/ios-sweet

python3 tools/mkpayload.py \
  /Users/ppkotik/Documents/unixnu-vayu/vayu-port/ci/kernel/kernel.vayu \
  shim/out/vayu.adt shim/out/payload-vayu-md0-gpu.bin \
  --ramdisk shim/out/sweet-init \
  --cmdline 'debug=0xc serial=3 cpus=1 keepsyms=1 -enable_kprintf_spam -noprogress rd=md0 wdt=-1 vayukeys=0 vayutouch=0 vayugpu=1 vayugpu_power=1 vayugpu_fw=1 sweetusb=0 sweetbattery=0 sweetlog=0'

CC='clang --target=aarch64-none-elf' LDFLAGS='-fuse-ld=lld' \
OBJCOPY=/usr/local/opt/llvm/bin/llvm-objcopy \
READELF=/usr/local/opt/llvm/bin/llvm-readelf \
NM=/usr/local/opt/llvm/bin/llvm-nm \
DTB=/Users/ppkotik/Documents/unixnu-vayu/uefi-vayu/Resources/DTBs/vayu.dtb \
DTB_OFFSET=0x2800000 \
OUT=out/shim-boot-vayu-md0-gpu.img \
./shim/build.sh ../shim/out/payload-vayu-md0-gpu.bin
```

## Known limitations / next steps

- The D22AP `/product` node is now in `research/ipsw-x/d22ap.adt` and in
  `shim/out/vayu.adt`.  This is the node the iOS `MobileGestalt`/activation
  path needs; it does not by itself make Apple activation succeed.  The
  framebuffer GUI path below is independent of activation.
- The vayu touch (Novatek NT36523 over SPI) and PM8150 volume-up key are
  ported and enabled in `shim-boot-vayu-md0-vayu-drv.img`; they still need
  hardware validation.  Charger/USB-PHY and Wi-Fi are not ported; the
  display uses the ABL splash framebuffer, not a DSI/GPU driver.  The Adreno
  640 has a stage-1 probe/power/GMU-boot driver (`vayu_gpu.c`, image
  `shim-boot-vayu-md0-gpu.img`) but no GX, CP, SMMU or iOS accelerator
  service yet.  The SM8150 references are in `../iPadOS-nabu`.
- The iPhone X 16.7.16 root-image pipeline is in `tools/mkiosroot-macos.sh`,
  the low-disk variant `tools/mkiosroot-lowdisk.sh`, and
  `tools/raw2simg.py`; see `VAYU-FLASH.md` for the build and flash steps.
- The kernel is still `MACHINE_CONFIG=SWEET`. A dedicated `VAYU` machine
  config can follow once the diagnostic boot confirms the memory,
  framebuffer, GIC/timer and UFS path. Renaming now would only add risk.
- For a diskless userspace test, a small HFS+ root with
  `tools/mkinit.py` can be passed to `mkpayload.py --ramdisk` and booted
  with `rd=md0`. The full iOS root belongs in `userdata` and needs explicit
  user approval before flashing.

## Reproduce the diagnostic image

```sh
cd /Users/ppkotik/Documents/unixnu-vayu/ios-sweet

python3 tools/mkadt.py shim/out/vayu.adt --vayu

python3 tools/mkpayload.py \
  xnu/out-0057/kernel.development.sweet shim/out/vayu.adt \
  shim/out/payload-vayu-diag.bin \
  --cmdline 'debug=0xc serial=3 cpus=1 keepsyms=1 -enable_kprintf_spam -noprogress rd=ufsroot0 wdt=-1 sweetkeys=0 sweettouch=0 sweetusb=0 sweetbattery=0'

CC='clang --target=aarch64-none-elf' LDFLAGS='-fuse-ld=lld' \
OBJCOPY=/usr/local/opt/llvm/bin/llvm-objcopy \
READELF=/usr/local/opt/llvm/bin/llvm-readelf \
NM=/usr/local/opt/llvm/bin/llvm-nm \
DTB=/Users/ppkotik/Documents/unixnu-vayu/uefi-sheng/Resources/DTBs/vayu.dtb \
OUT=out/shim-boot-vayu-diag.img \
./shim/build.sh ../shim/out/payload-vayu-diag.bin
```

The image layout is checked with:

```sh
python3 tools/unpack_bootv2.py shim/out/shim-boot-vayu-diag.img shim/out/check-vayu-diag
cmp shim/out/check-vayu-diag/ramdisk shim/out/payload-vayu-diag.bin
```

## vayu driver kernel (touch + keys), 2026-10-21

The vayu driver kernel is relinked locally from the 2026-10-18 19:25 CI
artifact (the `xnu/out-0057` directory name is historical; the artifact
inside is the latest SWEET build, through 0081); the source is under
`vayu-port/ci/buildtree/xnu/bsd/dev/`:

- `vayu_geni_spi.c/.h` — GENI SPI FIFO master (SE17, QUPV3_2)
- `vayu_nt36523.c/.h` — NT36xxx page/protocol helpers
- `vayu_touch.c` — vayu NT36xxx touch (SE17 `0xc80000`, wrapper `0xcc0000`,
  GCC RCG `0x1e148`, GPIO55/56/57/58 muxed to **qup17 = function index 1**,
  reset GPIO12, IRQ GPIO122, VDDIO GPIO100, L11C 3.0 V)
- `vayu_keys.c` — PM8150 SID0 GPIO6 volume-up, PON power/volume-down

`bsd/dev/sweet_ufs.c` calls `vayu_keys_init()` and `vayu_touch_init()`.
The old `sweet_*` input files remain linked but are not called.

Relink note: `link-vayu.sh` must **not** pass
`-Wl,-rename_segment,__LASTDATA_CONST,__LAST` (that is a static
kernel-cache flag). The stock development kernel keeps both
`__LASTDATA_CONST` and `__LAST`; `arm_vm_init.c` expects
`__LASTDATA_CONST` and `_last_kernel_symbol` at its usual address.

| File | Purpose | SHA-256 |
| --- | --- | --- |
| `shim/out/shim-boot-vayu-md0-vayu-drv.img` | `fastboot boot` driver test | `eb796623fb8019b9183121ce344782d5106b32bde876f9432ed8039bf311fa23` |
| `shim/out/payload-vayu-md0-vayu-drv.bin` | driver payload | `0814dad6d905c1ff19c090580b167eb761af533a7b818a8c58778884c27edfcb` |
| `vayu-port/ci/kernel/kernel.vayu` | relinked vayu kernel (stack-overflow fix, 2026-10-22) | `9d86ebdc26958fa1a74663756eca14a5d7facf9be8f4a1ed09a8cf3d289da268` |
| `vayu-port/ci/kernel/kernel.vayu.pre-stackfix` | previous relinked kernel (16 KiB `nvt_read_addr` frame) | `f3e0794b1b813f1161d63800c2c602d9f991e262317ce3450b5a3c83559a8d03` |
| `shim/out/shim-boot-vayu-md0-vayu-qemu.img` | QEMU/driver-off image | `51c665a9d3a75b7fbda224e27668ced315877c1143347d4142934d47cf665101` |
| `shim/out/shim-boot-vayu-ufs-vayu2.img` | full iOS-root boot image (`rd=ufsroot0`, debug console) | `14eb9dbbbe763ef311a57457a741f96f6c4a490fb53b7ffde9965d33aa209784` |
| `shim/out/shim-boot-vayu-ufs-vayu-gui2.img` | full iOS-root boot image (`rd=ufsroot0`, framebuffer GUI, `cpus=6`, no console) | `7ec920dae4f76c5901828bd7ad192bc7394999ea969abe68b33c521e1ba8eb36` |
| `shim/out/shim-boot-vayu-ufs-vayu4.img` | fixed full-root boot image (`rd=ufsroot0`, debug console), 2026-10-22 | `93c938f6769535586a47bf87ac426dc190c78db847ae9930b5fcd408c2364165` |
| `shim/out/payload-vayu-ufs-vayu4.bin` | payload for `vayu4` | `af32f5e6c240125940dd42dc2d1cc2813e59df1eccc1bd7c7d84bc375948e6df` |
| `shim/out/shim-boot-vayu-ufs-vayu-gui4.img` | fixed GUI image (`cpus=6`, framebuffer, no console), 2026-10-22 | `feab8a668fb5aa82e2c134a3f04468db00ed2ddd3025e79708738774efafc488` |
| `shim/out/payload-vayu-ufs-vayu-gui4.bin` | payload for `vayu-gui4` | `ba785757f87a7792e244627048b31511beff72928cf6f506324586f587f4895d` |
| `shim/out/shim-boot-vayu-ufs-vayu24.img` | **the beacon actually fires**: in `vayu23` the beacon lived inside `vayu_poll_hid()`, which only runs after the chip answered, so state 0 (the real state) produced no presses at all -- the build proved nothing.  `vayu24` fires it from the retry path every round and accepts `vayu_id_any()` (two identical reads of `0x3F004`, not all `0x00`/`0xFF`) next to the NT36523 trim table, so a live NT36672C is no longer rejected; GENI is tried before bit-bang again (it is the only transport that can carry the 139 KiB download) and bit-bang stays as the pads-are-alive fallback.  Beacon is volume-up, not power: power locks the phone and with touch dead nobody can unlock it.  Presses per burst = state+1: 1 = SPI silent, 2 = chip answers/no firmware, 3 = firmware alive, 4 = point frames, 2026-10-23 | `1da76095d6d71e8e1ac62bd9176e70cd2984a4a4bf8bc7d7684807089b51c9e8` |
| `shim/out/payload-vayu-ufs-vayu24.bin` | payload for `vayu24` (`vayutouch=1 vayutouchdelay=5 vayutouchpoll=16 vayutouchbang=1 vayuimg=24 sweetusb=0 sweetlog=0`) | `8c9700a123a4c423fa84152627579dd136b0104a1b89635d61a8b63ab6cef9ef` |
| `shim/out/shim-boot-vayu-ufs-vayu23.img` | **firmware download + visible state beacon**: if the event-page handshake fails the driver pushes the Huaxing `j20s_novatek_ts_fw02` image (139 KiB, embedded) exactly like the working Windows driver (eng reset, bootloader reset, ILM/DLM/info sections, CRC bank, BLD CRC, boot ready); there is no log channel, so the driver reports its state by pressing volume-up (volume HUD) `state` times every 15 s: 0 = SPI never answered, 1 = chip answered but no firmware, 2 = firmware alive (0xA5), 3 = point frames arriving, 2026-10-23 | `fc7d4ea70060aefa6b73350be6dc60f4721db8cb3dd8b6319f822a15f1edf2e7` |
| `shim/out/payload-vayu-ufs-vayu23.bin` | payload for `vayu23` (`vayutouch=1 vayutouchdelay=5 vayutouchpoll=16 vayutouchbang=1 vayuimg=23 sweetusb=0 sweetlog=0`) | `7cf88f660a5a2510c50902f53a7be928e9470b7670836d3f8d9e3f27ee53244b` |
| `shim/out/shim-boot-vayu-ufs-vayu22.img` | **NT36672C map + real software reset**: event page moved to `0x21C00` (0x2FE00 is the NT36523/nabu map), the bootloader reset `0x3F0FE=0x69` that parks the MCU without firmware is gone, replaced by the vendor handshake (clear `0x21C60`, `0x21C50=0xAE`, wait for `0xA5`); reset-line polarity is probed three ways (no reset / normal pulse / inverted), 2026-10-23 | `5898baee8aaadf8a76115f55195241367ca4b0d36f807ecc44ede559e35a97d7` |
| `shim/out/payload-vayu-ufs-vayu22.bin` | payload for `vayu22` (`vayutouch=1 vayutouchdelay=5 vayutouchpoll=16 vayutouchbang=1 vayuimg=22 sweetusb=0 sweetlog=0`) | `cdf2c8c0bd3a0ad5a8f654928cdf2ac36ec20cc85f63ce6b59a9bfb44da71d6b` |
| `shim/out/shim-boot-vayu-ufs-vayu21.img` | **self-calibrating** touch guess build: the chip's own trim id (`23 65 03`) is the oracle, so the driver sweeps bit-bang pad roles (6) x SPI mode 0/3 x read-dummy count (2/1/3/0) and then GENI speeds (9.6/4.8/1/0.4 MHz) x mode 0/3 x dummy, and keeps the combination that returns an NT36523 id; vendor framing and the old loose probe stay as the last resort, 2026-10-23 | `d186a674cdd07605ee2136a3d8d1874cad1422bde34e795340686ba6371587a8` |
| `shim/out/payload-vayu-ufs-vayu21.bin` | payload for `vayu21` (`vayutouch=1 vayutouchdelay=5 vayutouchpoll=16 vayutouchbang=1 vayuimg=21 sweetusb=0 sweetlog=0`) | `4846fb8edcda3100fc543622144fa8ec860d6ae86a56473a6fb2d7c4194ed0fa` |
| `shim/out/shim-boot-vayu-ufs-vayu20.img` | touch guess build (no diagnostics): **bit-banged SPI probed first** (`vayutouchbang=1`), GENI only as a fallback and only when the SE clocks work (clock failure is no longer fatal); reason: SE17 reports `FIFO_IF_DISABLE=1` (bootloader left it in GPI mode), so a "successful" GENI transfer proves the engine ran, not that the panel answered.  TLMM tile choice re-verified against mainline `pinctrl-sm8150.c` (gpio55-58/100/122 south, gpio12 north, `ctl=0x1000*id`, mux 2 / pull 0 / drive 6 / OE 9), 2026-10-23 | `3349f724062a864605bf9b4035966e11f60b8ed27b9a6ea6d9223bf51d444ee1` |
| `shim/out/payload-vayu-ufs-vayu20.bin` | payload for `vayu20` (`vayutouch=1 vayutouchdelay=5 vayutouchpoll=16 vayutouchbang=1 vayuimg=20 sweetusb=0 sweetlog=0`) | `4adcd748af309a32b423a1263a853ee87cb2286ccd954b926c7eb9cf9932f60c` |
| `shim/out/shim-boot-vayu-ufs-vayu19.img` | USB debug image: vayu18 died in stage 2, so stage 2 is split into four announced, paused steps (`2a` event buffer, `2b` DCFG/DEVTEN, `2c` ep0 commands, `2d` DALEPENA + setup TRB); `sweetusbskip2=1` keeps the bootloader's ep0 state instead, 2026-10-23 | `afa18288238a47cff608f49570a913be23a92774e470bf9d14acfd82f35e167b` |\n| `shim/out/payload-vayu-ufs-vayu19.bin` | payload for `vayu19` | `7295e4becf92e7383fac7d087bfaa62c0b77469fa4dedbe6ae787461d0ec7b74` |\n| `shim/out/shim-boot-vayu-ufs-vayu18.img` | **USB debug image, slowed down**: every USB hardware group prints `SWEET USB: #### ... ####` and then waits 6 s (`sweetusbpause=6000`), so the line left on screen when the phone resets names the group that kills it; same stages as vayu17 (`sweetusbstage=3`), touch off, 2026-10-23 | `1e0b1bc08dbf872e12f00ab1856812cd32a59b526c61b845fc5fa486ae727aa4` |\n| `shim/out/payload-vayu-ufs-vayu18.bin` | payload for `vayu18` | `d2d20401c125a65494ce3b1edfaa2104df78e1bdf24e0c0c380b49507061167b` |\n| `shim/out/shim-boot-vayu-ufs-vayu17.img` | **USB debug image** (`vayutouch=0`): the DWC3 gadget bring-up is split into printed stages and the DWC3 soft reset is off (`sweetusb=2 sweetusbstage=3 sweetusblog=1`); `SWEET USB: stage N ok` lines name the last step that survived, 2026-10-23 | `589ac3d04ea8af6582c883ebc1113dc8139d03a76f8c7b7cf840abe6b68c557b` |
| `shim/out/payload-vayu-ufs-vayu17.bin` | payload for `vayu17` (`vayuimg=17`, USB only) | `8bb846dfd483b792e57ba4bfbe90d309f0f7fbdd5445a08c4ddefee41a482279` |
| `vayu-port/ci/kernel/kernel.vayu` | relinked kernel for `vayu17` (staged `sweet_usb.c`) | `a4cbb02a67ca5078c5e19c49ce2b1ba657f7ccc9836d8ce6ba365063240ad2e9` |
| `shim/out/shim-boot-vayu-ufs-vayu16.img` | touch debug image: init takes the **HackiOS `redwood_goodix.c` recovery sequence** (cancel bit 2 -> abort bit 1, 200 ms per phase, RX FIFO drain, IRQ clear, never the DISABLE bit) and waits for `GENI_STATUS` bit 0 before every transfer; init stage trace kept, `SE_FIFO_IF_DISABLE` still opt-in (`vayutouchfix=2`), recovery off with `vayutouchfix=4`, 2026-10-23 | `0ca47aadc319bb87265dbd15e227cea7c3b41f684daafd3fda82f28605f4ffd6` |
| `shim/out/payload-vayu-ufs-vayu16.bin` | payload for `vayu16` (`vayuimg=16`) | `751f795ab5e2b2d1eca3e016e04767321d598314bd8e483bdc55ce862f832c04` |
| `vayu-port/ci/kernel/kernel.vayu` | relinked kernel for `vayu16` | `93e04a4ccde2645ce1659a0c921556a0aa79383c0321e0b301d5bf80cdbba4b1` |
| `shim/out/shim-boot-vayu-ufs-vayu15.img` | touch debug image: the `vayu13/14` freeze right after `geni info` is bounded -- the two repair steps added in vayu11/vayu12 (CANCEL/ABORT cleanup of a stuck command, `SE_FIFO_IF_DISABLE`=0x2008 clear) are now **off by default** (`vayutouchfix=1|2` turns them back on) and `geni_spi_init()` traces its stages (`geni init st / core / spi cfg / done`), 2026-10-23 | `27ba1abfa6c32f3247394d4e74ca97d7fd577091996d08d7542fbc0c02bb8b0b` |
| `shim/out/payload-vayu-ufs-vayu15.bin` | payload for `vayu15` (`vayuimg=15`, no `vayutouchfix`) | `9c58c725d33f88b166be0b670691ad56ebb1b170c6df19805f35362e1da28465` |
| `vayu-port/ci/kernel/kernel.vayu` | relinked kernel for `vayu15` | `04c8041a3e5a80000070270efe6460a49f1c983569d302fe99881496a0136878` |
| `shim/out/shim-boot-vayu-ufs-vayu14.img` | touch debug image: same as `vayu13` plus a build marker in the boot args (`vayuimg=14`) and an early `VAYU TOUCH: vayuimg=14 start` / `vayuimg=14 mapped ...` banner, north tile optional (reset skipped if it does not map), pin accessors are null-safe, 2026-10-23 | `eebda128200847daf12bc8b203bf419136ad97847ff064518ebb9009b87e40b2` |
| `shim/out/payload-vayu-ufs-vayu14.bin` | payload for `vayu14` (cmdline carries `vayuimg=14`) | `ba1cd3b5c88c8af91ca27be2eea642e7b5b7763a5f018fadd0da55665c91b500` |
| `vayu-port/ci/kernel/kernel.vayu` | relinked kernel for `vayu14` | `c9a710d98a7f93692fb42e545d9e695df568d83576e2485933a9d9d766787bc8` |
| `shim/out/shim-boot-vayu-ufs-vayu13.img` | touch debug image: the `vayu12` TLMM tile fix, but with **two small mappings** (south 512 KiB for gpio55-58/100/122, north 64 KiB for gpio12) instead of one 14 MiB one -- `vayu12` panicked in `kmem_alloc(..., 14680064, ...) failed with 3 @vm_kern.c:178`; every pin access now goes through `vayu_pin_*()` helpers, 2026-10-22 | `5ed158044eb95089bd4697b6534c99d79f275d96aa518fdfed0604dc6dd4e741` |
| `shim/out/payload-vayu-ufs-vayu13.bin` | payload for `vayu13` | `4f73fe69948943d2c34557b59bad5721497100ded045d2b9d72403d524ffb4f7` |
| `vayu-port/ci/kernel/kernel.vayu` | relinked kernel for `vayu13` (TLMM south/north tiles mapped separately, FIFO interface reclaim, GENI diagnostics) | `2d42445665b1c8e3780beffbd2628c4aa64129987783184923717f8ee0a61f2d` |
| `shim/out/shim-boot-vayu-ufs-vayu12.img` | **panics at boot, do not boot** (`kmem_alloc` of the 14 MiB TLMM mapping fails): touch debug image: **TLMM tile fix** (SM8150 pads live in four 3 MiB tiles at 0x3100000/0x3500000/0x3900000/0x3d00000, not at 0x3000000+0x1000*pin, so every pin write went into a hole and every read returned 0), `SE_FIFO_IF_DISABLE` (0x2008) cleared when the bootloader left the SE in GPI mode, plus the `vayu11` GENI diagnostics, tile-map check line, 2026-10-22 | `a9f97a86c70f578938e736f708b13a5400afcb4ef54ceab9e01babf2a9c48c70` |
| `shim/out/payload-vayu-ufs-vayu12.bin` | payload for `vayu12` | `b3e04840e2e77ca05ffd54249ae74504293fd585a5b45fb88f378feafff587d4` |
| `vayu-port/ci/kernel/kernel.vayu` | relinked kernel for `vayu12` (TLMM tile map, FIFO interface reclaim, GENI diagnostics) | `b7f726ee2f3258afc90f2dd29eb7b130578aada2b643c66bf19d8b60412e9ff2` |
| `shim/out/shim-boot-vayu-ufs-vayu11.img` | touch debug image: clears a command the bootloader may have left running in the SE (`M_GENI_CMD_ACTIVE`), keeps the real `GENI_ERR_*` code instead of the collapsed `-1`, records whether the sequencer ever started, report/DIGEST gain `geni=/act=/err=`, 2026-10-22 | `cc670cd66775021d4bb94f36c59f6ce9175454e8c34513d2073a2989e2645c90` |
| `shim/out/payload-vayu-ufs-vayu11.bin` | payload for `vayu11` | `e027dd493355cbcca97cada67be819cf378692fa3f3fe1541a51adeb2c575209` |
| `vayu-port/ci/kernel/kernel.vayu` | relinked kernel for `vayu11` | `9259b774bdae84886351c5380b22f4cebd94918484ee303410550263577a70bb` |
| `shim/out/shim-boot-vayu-ufs-vayu10.img` | touch debug image: GENI internal loopback self-test, bit-banged SPI search over all six MOSI/MISO/CLK pad assignments, 20 ms reset pulse, IRQ-line float test, one-shot report page, 2026-10-22 | `2fc850ff26a3bbfa09c837ab19f519db2f5a67ff8ddd5bc61222d2de899208df` |
| `shim/out/payload-vayu-ufs-vayu10.bin` | payload for `vayu10` | `a5ae4d7e45eea04f51fcf3d7b750a707698329f9b343d91a957a8ba222016883` |
| `shim/out/shim-boot-vayu-ufs-vayu9.img` | touch debug image: IRQ-line float test, RCG on the 19.2 MHz crystal, GENI at 9.6/4.8/1/0.4 MHz, bit-banged SPI fallback, tolerant trim-id check, and a one-shot `==== VAYU TOUCH REPORT ====` page after which the console stays still, 2026-10-22 | `d23499336f87656e2380928fd2c4d11f263810d91e7b7b9b76b30cb426878d5e` |
| `shim/out/payload-vayu-ufs-vayu9.bin` | payload for `vayu9` | `e66efa3fd0f3bcf4b43085a2c770b76074c07fff843a745159fa6059cd5d1a3f` |
| `shim/out/shim-boot-vayu-ufs-vayu8.img` | **do not use**: same diagnostics plus `sweetusb=2 sweetusblog=1`; the USB gadget resets the phone on vayu (no logs reach the host) | `f05bf3561529921816b697486c9e14a7358a4f1023f2889bd4f8b664ed26d41d` |
| `shim/out/payload-vayu-ufs-vayu8.bin` | payload for `vayu8` | `d050770a7904761a45121039c464f885351c16596c47045c630e65b3f6a0f45a` |
| `shim/out/shim-boot-vayu-ufs-vayu7.img` | touch debug image: RCG on the 19.2 MHz crystal, GENI probe at 9.6/4.8/1/0.4 MHz, bit-banged SPI fallback, tolerant trim-id check (Huaxing/CSOT units), persistent `RESULT`/`STATUS` lines and poll heartbeat, 2026-10-22 | `75448a0cd2274f624b97c42c5f6388f174f5e18c222908afee0e7c4ae35d45dd` |
| `shim/out/payload-vayu-ufs-vayu7.bin` | payload for `vayu7` | `20bc3e9f1abab919b96e46310320c9994c075a703c694745dfe3712fc77b95dc` |
| `shim/out/shim-boot-vayu-ufs-vayu-gui7.img` | same kernel, GUI (`cpus=6`, `sweetfb_noconsole=1`), 2026-10-22 | `3bdb5554593104c863ad01312357cc1148d6ecb6816c5c39f7632d8443390348` |
| `shim/out/shim-boot-vayu-ufs-vayu6.img` | superseded touch debug image (`01e22b77…`), 2026-10-22 | `01e22b77dae61b4351f97c7d86ad7f6c642730573693dd33f4c2bf2c964562ec` |
| `shim/out/payload-vayu-ufs-vayu6.bin` | payload for `vayu6` | `4f7776703c522398f032d64dbd007e56b266d4ed1324ea24a3e965080cda3e03` |
| `shim/out/shim-boot-vayu-ufs-vayu-gui6.img` | same kernel, GUI (`cpus=6`, `sweetfb_noconsole=1`), 2026-10-22 | `bf5c0fe5bd1fb89b0ee57ba3eae057136395da982d4acd577d30a56c9f0707da` |
| `shim/out/payload-vayu-ufs-vayu-gui6.bin` | payload for `vayu-gui6` | `bba753d1b586560bdd957891910cb6f66bd7ad6a311fe746320de5d9356208db` |
| `vayu-port/ci/kernel/kernel.vayu` | relinked kernel for `vayu10` (touch clock fix, GENI loopback test, bit-bang role search, IRQ float test, one-shot report page), 2026-10-22 | `62b5cd33fd9d4b662c43785bd2060da3ab91d4b6346963769e70307e1f5074a0` |
| `shim/out/shim-boot-vayu-ufs-vayu5.img` | superseded touch debug image (no bit-bang fallback) | `94616d2bd1a8a89ece67a45972976b4cb4db4eae843ad4e20207ae4d5779a0ee` |
| `vayu-ios-out/ios16-root.img` | final iOS root, raw HFS+ (16G logical, 11G actual) | `89f4f974a0b8e0c0038dbc0790df961d0e854313b5d3285631a56e98f9b8f4da` |
| `shim/out/shim-boot-vayu-md0-gpu.img` | GPU stage-1 probe image (`vayugpu=1 vayugpu_power=1 vayugpu_fw=1`), 2026-10-22 | `7eb9905880e6258f6380bb359496e17566252f50c653816e1796799b15fbed01` |
| `shim/out/payload-vayu-md0-gpu.bin` | payload for `vayu-md0-gpu` | `0b0d89b37bb2ed31fccb244594910d3c022acb6d9f09deba72a0c684aca0ca16` |
| `vayu-port/ci/kernel/kernel.vayu` | relinked kernel with `vayu_gpu.c` (GPU stage 1), 2026-10-22 | `af551756905f3ad35992c999a73a1316afc60c0d12b30513eb9d870ef6aaedea` |
| `shim/out/shim-boot-vayu-md0-gpuprobe.img` | GPU probe-only image (`vayugpu=1`, no CX power), 2026-10-22 | `4a5aa1ace939fb22b8fb56ca955f48253b17103dd4ed44ebeca409913d981e30` |
| `shim/out/payload-vayu-md0-gpuprobe.bin` | payload for `vayu-md0-gpuprobe` | `283c93d10d2fe99aa99cc1ab481f190ecec2e407441464a75f87220393d37d51` |

Build the driver image with:

```sh
cd /Users/ppkotik/Documents/unixnu-vayu/ios-sweet

python3 tools/mkpayload.py \
  /Users/ppkotik/Documents/unixnu-vayu/vayu-port/ci/kernel/kernel.vayu \
  shim/out/vayu.adt shim/out/payload-vayu-md0-vayu-drv.bin \
  --ramdisk shim/out/sweet-init \
  --cmdline 'debug=0xc serial=3 cpus=1 keepsyms=1 -enable_kprintf_spam -noprogress rd=md0 wdt=-1 vayukeys=1 vayutouch=1 vayutouchdelay=5 vayutouchpoll=8 sweetusb=0 sweetbattery=0 sweetlog=0'

CC='clang --target=aarch64-none-elf' LDFLAGS='-fuse-ld=lld' \
OBJCOPY=/usr/local/opt/llvm/bin/llvm-objcopy \
READELF=/usr/local/opt/llvm/bin/llvm-readelf \
NM=/usr/local/opt/llvm/bin/llvm-nm \
DTB=/Users/ppkotik/Documents/unixnu-vayu/uefi-sheng/Resources/DTBs/vayu.dtb \
DTB_OFFSET=0x2800000 \
OUT=out/shim-boot-vayu-md0-vayu-drv.img \
./shim/build.sh ../shim/out/payload-vayu-md0-vayu-drv.bin
```

`DTB_OFFSET` must stay above `0x1000000 + payload_size`; the old
`0x2200000` is too low for the 19 MB vayu payload and would overwrite the
vayu DTB.

Boot it non-destructively:

```sh
fastboot boot /Users/ppkotik/Documents/unixnu-vayu/ios-sweet/shim/out/shim-boot-vayu-md0-vayu-drv.img
```

Expected after the normal kernel/`md0` lines and the init heartbeat:

```
VAYU KEYS: arb=... apids=... pon-apid=... gpio6-apid=...
VAYU KEYS: key=... down=...
VAYU TOUCH: probe starting in 5 s
VAYU TOUCH: L11C vote=0
VAYU TOUCH: geni info ...
VAYU TOUCH: trim @0x3f004 ...
VAYU TOUCH: down id=1 x=... y=...
```

The driver test is RAM-only (`rd=md0`); it does not mount or write
`userdata`. This is still a DEBUG/development kernel, not RELEASE.

## Kernel stack overflow in the touch probe (fixed 2026-10-22)

The boot after the launchd fix did not reach SpringBoard: ~5 s in it
panicked with (`vayu3dbg2`, decoded against `kernel.vayudbg2`):

```
panic(cpu 0 caller 0xfffffe0007ad5508): Invalid kernel stack pointer
(probable overflow), at pc 0xfffffe0007a96820, lr 0xfffffe0007a97d9c
  caller 0xfffffe0007ad5508 = panic_with_thread_kernel_state+0x124
  pc     0xfffffe0007a96820 = geni_spi_transfer+0x4
  lr     0xfffffe0007a97d9c = nvt_read_addr+0xa8
  esr 0x96000047 -> data abort, level-3 translation fault on a stack write
  sp 0xfffffe8fa0b57b10, far 0xfffffe8fa0b57bf0 (= sp+0xe0) -> guard page
```

`nvt_read_addr()` allocated `uint8_t tx[8194]; uint8_t rx[8194];` on the
stack (16,388 bytes, inherited from the nabu reference).  Clang emitted
`mov w9,#0x4010; bl __chkstk; sub sp,sp,#0x4000`, which cannot fit a 16 KiB
arm64 kernel stack; the first SPI read after `vayutouchdelay=5` pushed sp
into the guard page and the frame store faulted.  This is why the phone
looked like it booted: the panic is 5 s after the touch thread starts, not
at launchd.

Fix: both staging buffers are now `NVT_XFER_MAX + 2` (256 + 2) bytes and
`nvt_read_addr()` rejects `len > 256`; the frame is 0x220 bytes.  No caller
reads more than 65 bytes (point frame) or 39 bytes (FWINFO).

Build (the kernel object then the relink; the compile flags must match the
CI SWEET build):

```sh
cd /Users/ppkotik/Documents/unixnu-vayu/vayu-port/ci
./buildtree/compile-one.sh \
  buildtree/xnu/bsd/dev/vayu_nt36523.c kernel/objects/vayu_nt36523.o
cd kernel && ./link-vayu.sh
```

Verify the fix in the linked kernel (must be `sub sp, sp, #0x220`, not
`#0x4000`):

```sh
xcrun llvm-objdump -d --start-address=0xfffffe0007a97834 \
  --stop-address=0xfffffe0007a97890 kernel.vayu | head
```

Then rebuild payload + image exactly as above (same `--cmdline`,
`DTB=uefi-sheng/Resources/DTBs/vayu.dtb`, `DTB_OFFSET=0x2800000`).  The
rebuilt boot image is byte-identical to the old one when given the old
payload, which validates the repack.

## iPhone X 16.7.16 root image (macOS)

The iPhone X `iPhone10,3,iPhone10,6_16.7.16_20H392_Restore.ipsw` contains
the three images the root needs.  `tools/ipsw-fetch.py` reads the ZIP64
central directory over HTTP and downloads only those entries:

| BuildManifest key | IPSW entry | stored / inflated |
| --- | --- | --- |
| `OS` | `087-86698-044.dmg` | 4,730,814,044 (UDIF) |
| `Cryptex1,SystemOS` | `087-86593-047.dmg` | 1,097,727,251 -> 2,743,074,816 (raw APFS) |
| `Cryptex1,AppOS` | `087-86715-047.dmg` | 2,381,197 -> 10,485,760 (raw APFS) |

The BuildManifest names the entries; `--list` prints them.  Example:

```sh
URL='https://updates.cdn-apple.com/2025FallFCS/fullrestores/122-77006/1158E502-D76D-4226-AA06-ED5E30CD1111/iPhone10,3,iPhone10,6_16.7.16_20H392_Restore.ipsw'
python3 tools/ipsw-fetch.py --list "$URL"
python3 tools/ipsw-fetch.py "$URL" 087-86698-044.dmg OS.dmg
python3 tools/ipsw-fetch.py "$URL" 087-86593-047.dmg SystemOS.dmg
python3 tools/ipsw-fetch.py "$URL" 087-86715-047.dmg AppOS.dmg
```

The D22AP `/product` node comes from the IPSW's
`Firmware/all_flash/DeviceTree.d22ap.im4p`; it is saved as
`research/ipsw-x/d22ap.adt` (182 KiB).  `tools/mkadt.py --vayu` now picks it
up automatically, so the vayu ADT identifies as iPhone X while keeping the
sweet driver names.

Build the case-sensitive journaled HFS+ image (run as root; it writes a new
file and preserves iOS numeric owners).  With all three DMGs present:

```sh
sudo tools/mkiosroot-macos.sh OS.dmg SystemOS.dmg AppOS.dmg ios16-root.img 16G
```

On a low-disk host fetch and merge one DMG at a time; this is the flow used
for the internal build:

```sh
sudo tools/mkiosroot-lowdisk.sh init OS.dmg ios16-root.img 16G
rm OS.dmg
sudo tools/mkiosroot-lowdisk.sh add ios16-root.img SystemOS.dmg OS
rm SystemOS.dmg
sudo tools/mkiosroot-lowdisk.sh add ios16-root.img AppOS.dmg App
rm AppOS.dmg
```

`OS.dmg` is a UDIF (APFS inside); `SystemOS.dmg`/`AppOS.dmg` are raw APFS.
macOS attaches them with `hdiutil`; the cryptexes go to
`/private/preboot/Cryptexes/{OS,App}`, the dyld cache is hard-linked, and
`tools/sweet-root-tweaks.py` applies the bring-up edits.  The raw 16 GiB
HFS+ image uses about 8 GiB after the OS volume and about 10-11 GiB after
both cryptexes.  `raw2simg.py` makes an Android sparse image (~12 GiB);
only do that on a disk with room for both files.

Flash the raw image (this **erases Android user data**; the internal build
does not keep the sparse copy).  Use the bundled platform-tools 32.0.0
fastboot, not the installed 37: fastboot 33+ reads the entire 16 GiB raw
image into memory for its AVB-footer check and dies with
`fastboot: error: Failed reading from userdata`.  Version 32.0.0 only
does that check for `boot`.

```sh
cd /Users/ppkotik/Documents/unixnu-vayu
FB=tools/fastboot32
$FB -S 512M flash userdata vayu-ios-out/ios16-root.img
$FB boot ios-sweet/shim/out/shim-boot-vayu-ufs-vayu2.img
```

`-S 512M` sparsifies the raw image on the fly, so no second 12 GiB file is
needed.  See `VAYU-FLASH.md` for the full pre-flight and recovery steps.

The `shim-boot-vayu-ufs-vayu2.img` image uses `rd=ufsroot0` and no ramdisk.
For a RAM-only smoke test use `shim-boot-vayu-md0-vayu-drv.img` instead.
The final kernel is still DEBUG/development, not RELEASE.

