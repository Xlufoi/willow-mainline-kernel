# willow-mainline-kernel

Mainline Linux for the **Xiaomi Redmi Note 8 / Note 8T** — Qualcomm **SM6125**
(SoC codename *trinket*). The Xiaomi codenames are **ginkgo** (Note 8) and
**willow**/(8T); they are the same board family with different panels/NFC, so
this tree covers both.

This is a personal bring-up tree, not something you should merge into mainline.
It is based on the upstream **Linux 7.0** tag, with the ongoing work to get
audio and the rest of the phone running.

![status: boots](https://img.shields.io/badge/boots-yes-brightgreen)
![audio: not yet](https://img.shields.io/badge/speaker-silent-red)

## What works, what doesn't

| Area | Status |
|------|--------|
| Boot to userspace (systemd on eMMC) | ✅ |
| UART serial console | ✅ |
| Display (DSI) — Tianma NT36672A | ✅ |
| Display (DSI) — Huaxing FT8719 | ✅ (touch still off) |
| SPI touch (NT36672A) | ✅ |
| USB gadget — RNDIS + SSH | ✅ |
| Wi-Fi (WCN3990 / `ath10k_snoc`) | ✅ |
| Adreno 610 + GNOME desktop | ✅ |
| Docker | ✅ |
| **Speaker audio (MAX98937)** | ❌ silent |
| **3.5 mm jack (WCD9375 / SoundWire)** | ❌ not described in DT |
| Camera | ❌ |

The display, Wi-Fi, GPU and USB work well enough to use the phone as a small
Linux box over SSH. Audio is the thing I'm actively poking at — see below for
exactly how far it got, so nobody has to redo the detective work.

## Install

Prebuilt images are attached to the [Releases](../../releases) page. You need
an unlocked bootloader and `fastboot`.

```sh
fastboot getvar product          # must print: ginkgo
fastboot flash dtbo dtbo-empty.img
fastboot flash boot boot.img     # pick the image for your panel
fastboot reboot
```

The dtbo only needs flashing once — it clears the stock Android overlays so the
mainline DTB is actually used. The root filesystem is not in this repo; this
tree is just the kernel side.

## Build

Out-of-tree build, clang/LLD:

```sh
make O=../out/kernel ARCH=arm64 LLVM=1 LLVM_IAS=1 defconfig
make O=../out/kernel ARCH=arm64 LLVM=1 LLVM_IAS=1 -j"$(nproc)" Image.gz dtbs modules
```

## The audio story (a.k.a. why is the speaker still silent)

tl;dr: **the clocks and the data are fine, the amp just won't sing.**

I traced the whole path — front-end PCM → ADM → AFE → MI2S → MAX98937 — and got
as far as proving the digital side is alive:

- The DSP routes the stream (ADM matrix map succeeds, AFE port `0x1000` starts).
- BCLK and WS are physically toggling on the MI2S pads (measured on the TLMM
  with a small sampler module: GPIO113 = bit clock, GPIO114 = frame sync).
- I2S data is present on the data line (GPIO116, ~48 kHz frame rate).
- The amp is out of reset (GPIO27 high) and talks over I²C.

The catch: the part on this board is a **MAX98937 rev B**, but the mainline
`max98927` driver assumes the older MAX98927 register map. Several output-path
registers moved or changed meaning, so the driver was quietly writing to the
wrong places:

| Register | MAX98927 | MAX98937 rev B |
|----------|----------|----------------|
| `0x20`/`0x21` | PCM mode / master mode | *status word config* |
| `0x23`/`0x24`/`0x25` | — | PCM mode / master mode / clock setup |
| `0x3a` | `AMP_EN` | `Speaker_source_select` |
| `0x3b` | `SPK_SRC_SEL` | `Speaker_Gain` |
| `0x3c` | `SPK_GAIN` | `SSM_Configuration` |

Two of these were actively breaking output: DAPM wrote `0x3a` bit0 (which on
rev B selects an *invalid* speaker source instead of "enable"), and `set_fmt()`
clobbered the speaker gain by treating `0x3b` as the source-select register.
This tree fixes both and programs the downstream rev B init table, so the amp
register set now matches what the vendor driver leaves behind:

```
0x3a=0x00  0x3b=0x55  0x3c=0xa0  0x39=0x81  0x0500=0x01
```

…and it still doesn't make a sound. The amp's interrupt-status register reports
`PWRUP_DONE` but never latches `CLKSTART`, so the next suspect is the amp's own
clock/BSEL configuration or a boost/SSM detail. If you know MAX98937 rev B,
[open an issue](../../issues) — I'd love a second pair of eyes.

## Layout / provenance

- This repository is the kernel source tree (Linux 7.0 + the bring-up patches).
  It is a squashed snapshot, so it does not carry the full upstream history —
  grab upstream separately if you need `git log`.
- The device tree for the panel variant is under
  `arch/arm64/boot/dts/qcom/`.
- Audio changes live in `sound/soc/codecs/max98927.c` and
  `sound/soc/qcom/qdsp6/`.

Everything here is GPL-2.0-only, same as the kernel.
