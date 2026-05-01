# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Linux 5.4 kernel for the Motorola **miami** device (Moto G51 5G) based on the Qualcomm SM6375 (internally called **holi**) SoC. Used as the kernel for LineageOS builds. This is a personal fork (`origin`: ast261/android_kernel_motorola_sm6375) tracking `upstream` (LineageOS/android_kernel_motorola_sm6375), which supports additional holi devices — but only miami is built here.

## Branch naming

- `lineage-X.Y` — base LineageOS kernel branch
- `lineage-X.Y-ksun` — + KernelSU Next (kernel root manager)
- `lineage-X.Y-ksun-X.Y.Z-susfs-X.Y.Z` — + KernelSU Next + SusFS

## Building

### Prerequisites

Requires Clang/LLVM toolchain (`LLVM=1` is mandatory). The build.config system expects prebuilts at `prebuilts-master/clang/host/linux-x86/clang-r416183b/bin` relative to the kernel dir (set `CLANG_PREBUILT_BIN` or ensure `clang` is in `PATH`).

### Standalone make (recommended for development)

```bash
# Generate defconfig for holi QGKI variant
make ARCH=arm64 LLVM=1 O=out vendor/holi-qgki_defconfig

# Apply Motorola holi fragment (used for device-specific builds)
# scripts/kconfig/merge_config.sh -m out/.config arch/arm64/configs/vendor/ext_config/moto-holi.config

# Build
make ARCH=arm64 LLVM=1 O=out -j$(nproc) Image.gz dtbs modules
```

Output lands in `out/arch/arm64/boot/` (Image.gz) and `out/arch/arm64/boot/dts/vendor/qcom/` (DTBs).

### Using the AOSP build.config system

```bash
BUILD_CONFIG=build.config.msm.lahaina VARIANT=qgki build/build.sh
```

Variants: `qgki-debug`, `qgki-consolidate`, `qgki`, `gki`, `gki-only`.

### menuconfig

```bash
make ARCH=arm64 LLVM=1 O=out menuconfig
```

The build.config system wraps this via a shell function in `build.config.msm.common` that saves back to the defconfig fragment.

## Key config locations

| File | Purpose |
|---|---|
| `arch/arm64/configs/vendor/holi-qgki_defconfig` | Main defconfig for holi QGKI builds |
| `arch/arm64/configs/vendor/ext_config/moto-holi.config` | Motorola-specific base fragment |
| `arch/arm64/configs/vendor/ext_config/moto-holi-miami.config` | miami device fragment (primary target) |
| `arch/arm64/configs/vendor/ext_config/lineage_moto-holi.config` | LineageOS-specific fragment |

## Code architecture

### Platform-specific code

- `techpack/` — Qualcomm out-of-tree modules: audio, camera, display, video, dataipa, datarmnet. Compiled conditionally via `TECHPACK` flag.
- `drivers/staging/qcacld-3.0/` and `qca-wifi-host-cmn/` — Qualcomm WiFi stack (`wlan.ko`, `CONFIG_QCA_CLD_WLAN=m` in `lineage_moto-holi.config`; large, mostly self-contained).
- `drivers/phy/motorola/` — Motorola USB PHY drivers.
- `arch/arm64/boot/dts/qcom/` — Device trees (no Motorola-specific DTS in this repo; those live in a separate device tree repo).

Miami kernel modules (`moto-holi-miami.config`):

*Camera*
- `drivers/media/platform/msm/camera/cam_sensor_module/cam_cci/` — Camera CCI interface (`cci_intf.ko`, `CONFIG_CAMERA_CCI_INTF=m`).

*Regulators*
- `drivers/regulator/wl2868c/` — WL2868C PMIC regulator (`wl2868c.ko`, `CONFIG_REGULATOR_WL2868C=m`).
- `drivers/regulator/wl2864c/` — WL2864C PMIC regulator (`wl2864c.ko`, `CONFIG_REGULATOR_WL2864C=m`).

*Power / charging*
- `drivers/power/qpnp_adaptive_charge/` — Adaptive charge control (`qpnp_adaptive_charge.ko`, `CONFIG_QPNP_ADAPTIVE_CHARGE=m`).
- `drivers/power/mm8013c_fg_mmi/` — MM8013C fuel gauge (`mm8013c_fg_mmi.ko`, `CONFIG_MMI_MM8013C_FG=m`).
- `drivers/power/mmi_charger/` — Motorola charger base class (`mmi_charger.ko`, `CONFIG_MMI_CHARGER=m`).
- `drivers/power/mmi_discrete_charger/` — Discrete charger driver (`mmi_discrete_charger.ko` + `mmi_discrete_charger_class.ko`, `CONFIG_MMI_DISCRETE_CHARGER/CLASS=m`).
- `drivers/power/mmi_discrete_turbo_charger/` — Turbo / pump charger policy (`mmi_discrete_turbo_charger.ko`, `CONFIG_MMI_DISCRETE_TURBO_CHARGER=m`).
- `drivers/power/sgm4154x_chg_mmi/` — SGM4154X charger (`sgm4154x_charger.ko`, `CONFIG_SGM4154X_CHARGER=m`).
- `drivers/power/bq2589x_chg_mmi/` — TI BQ2589X charger (`bq2589x_charger.ko`, `CONFIG_BQ2589X_CHARGER=m`).
- `drivers/power/bq25980_mmi_iio/` — BQ25980 flash charger IIO (`bq25980_mmi_iio.ko`, `CONFIG_BQ25980_MMI_IIO_CHARGER=m`).
- `drivers/power/cps4019_wls_charger/` — CPS4019 wireless charger (`cps4019_wls_charger.ko`, `CONFIG_CPS_WLS_CHARGER=m`).

*USB Type-C*
- `drivers/usb/typec/adapter_class/` — USB PD adapter class (`adapter_class.ko`, `CONFIG_TYPEC_ADAPTER_CLASS=m`). `mmi_tcpc` imports its symbols via `Module.symvers`.
- `drivers/usb/typec/mmi_tcpc/` — Motorola TCPC USB PD stack: `tcpc_class.ko`, `tcpc_rt1711h.ko`, `rt_pd_manager.ko` (`CONFIG_TCPC_CLASS/RT1711H/RT_PD_MANAGER=m`).

*Input*
- `drivers/input/misc/rbs_fod_mmi/` — Fingerprint-on-display input (`rbs_fod_mmi.ko`, `CONFIG_INPUT_RBS_FOD_MMI=m`).
- `drivers/input/touchscreen/goodix_berlin_mmi/` — Goodix BRL touchscreen (`goodix_brl_mmi.ko`, `CONFIG_TOUCHSCREEN_GOODIX_BRL_MMI=m`).
- `drivers/input/touchscreen/touchscreen_mmi/` — Touchscreen MMI class/panel abstraction (`touchscreen_mmi.ko`, `CONFIG_INPUT_TOUCHSCREEN_MMI=m`). Imports `sensors` and `mmi_relay` symbols.

*LEDs / haptics*
- `drivers/leds/aw2033/` — AW2033 RGB LED driver (`leds_aw2033.ko`, `CONFIG_LEDS_AW2033=m`).
- `drivers/misc/awinic/aw862x/` — Awinic AW862X haptic driver (`aw862x.ko`, `CONFIG_AW862X_HAPTIC=m`).

*Motorola MMI framework*
- `drivers/mmi_annotate/` — Panic annotation (`mmi_annotate.ko`, `CONFIG_MMI_ANNOTATE=m`).
- `drivers/mmi_info/` — Device info: storage, RAM, unit, boot (`mmi_info.ko`, `CONFIG_MMI_INFO=m`).
- `drivers/mmi_relay/` — Relay driver (`mmi_relay.ko`, `CONFIG_MMI_RELAY=m`).
- `drivers/misc/mmi_sys_temp/` — System temperature monitor (`mmi_sys_temp.ko`, `CONFIG_MMI_SYS_TEMP=m`).
- `drivers/misc/utag/` — UTAG persistent storage (`utags.ko`, `CONFIG_MOTO_UTAGS=m`).
- `drivers/sensors/` — Sensors class abstraction (`sensors_class.ko`, `CONFIG_SENSORS_CLASS=m`).

*NFC / proximity*
- `drivers/nfc/sn1xx/` — NXP SN1xx NFC (`nfc_i2c.ko`, `CONFIG_NFC_SN1XX=m`).
- `drivers/misc/sx937x/` — SX937x SAR / proximity sensor (`sx937x_sar.ko`, `CONFIG_SARSENSOR_SX937X=m`).

*Filesystem*
- `fs/exfat/` — ExFAT filesystem (`exfat.ko`, `CONFIG_EXFAT_FS=m` in `lineage_moto-holi.config`).

### Build system layering

Build configs are sourced in order:
1. `build.config.common` (LLVM, Android flags)
2. `build.config.aarch64` (arm64 arch, output files)
3. `build.config.msm.lahaina` (MSM_ARCH=lahaina, VARIANT selection)
4. `build.config.msm.common` (DTB support, defconfig generation)
5. `build.config.msm.gki` (DEFCONFIG path: `vendor/${MSM_ARCH}-${VARIANT}_defconfig`)

### GKI defconfig generation

The defconfig at `arch/arm64/configs/vendor/holi-qgki_defconfig` is auto-generated by `scripts/gki/generate_defconfig.sh` by merging GKI base + holi QGKI + moto-holi fragments. Do not edit it directly; edit the source fragments instead.

## Patch requirements

All commits must follow Android Common Kernel conventions (from README.md):

- Subject prefix: `UPSTREAM:` (cherry-pick from mainline), `BACKPORT:` (modified mainline), `FROMGIT:` (from maintainer tree), `FROMLIST:` (from mailing list), or `ANDROID:` (Android-specific)
- `Signed-off-by:` required
- Patches must pass `scripts/checkpatch.pl`
- Must not break `gki_defconfig` or `allmodconfig` builds

```bash
scripts/checkpatch.pl --strict -g HEAD
```

## Remotes

- `origin` — personal fork (push target): `git@github.com:ast261/android_kernel_motorola_sm6375.git`
- `upstream` — LineageOS official (fetch only): `https://github.com/LineageOS/android_kernel_motorola_sm6375.git`
