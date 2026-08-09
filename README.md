# Obsidian UI
<img width="1254" height="1254" alt="ObsidianUI" src="https://github.com/user-attachments/assets/df1e5369-bd28-4b4f-9b48-86a19bd9daf6" />

**Obsidian UI** is a bash-based pipeline that ports a Samsung "stock" device
configuration (build props, `floating_feature.xml`, SELinux policy, VNDK,
debloat rules, framework smali patches) onto another device's firmware, then
rebuilds flashable partition images. This branch, **fifteen**, targets
firmware built on **Android 15 / One UI 7**.

Give it a firmware download link for `TARGET_DEVICE` and a known-good config
for `STOCK_DEVICE`, and it produces a "ported" build: debloated, SELinux
patched, framework-patched (secure folder, screenshot restrictions, flag
secure, Bluetooth compatibility), and stamped with Obsidian UI branding
instead of the target's original Samsung build ID.

> ⚠️ This is a low-level firmware modification tool: it downloads, unpacks,
> decompiles/recompiles system components, patches SELinux policy, and
> repacks Samsung firmware images. Use it only on devices and firmware you
> own or are authorized to modify, and keep a way to restore stock firmware
> before flashing anything built with this tool.

---

## How it works

```
EXTRACT_SUPER_IMG  →  EXTRACT_FIRMWARE_IMG  →  DECODE_OMC
        │
        ▼
DEBLOAT  →  APPLY_STOCK_CONFIG (VNDK, system_ext layout, camera/eSIM fixups)
        │
        ▼
PATCH_SELINUX  →  DISABLE_SECURITY  →  ADD_SAMSUNG_FLAGSHIP_APPS
        │
        ▼
APPLY_CUSTOM_FEATURES (floating features, build.prop tweaks)
        │
        ▼
DECOMPILE services.jar / ssrm.jar / samsungkeystoreutils.jar (apktool)
        │
        ▼
PATCH_SSRM  →  PATCH_FLAG_SECURE  →  PATCH_SECURE_FOLDER  →  PATCH_PRIVATE_SHARE
        │
        ▼
RECOMPILE jars  →  PATCH_BT_LIB (hex patch)
        │
        ▼
Set ro.build.display.id → "... Built with Obsidian UI"
        │
        ▼
BUILD_IMG (erofs / ext4 / f2fs)  →  OUT/*.img
```

All of the above is orchestrated by [`fifteen.sh`](fifteen.sh) and also runs
automatically in CI via [`.github/workflows/fifteen.yml`](.github/workflows/fifteen.yml).

### Key terms

| Term | Meaning |
|---|---|
| **STOCK_DEVICE** | The device whose *configuration* you want applied (build props, floating features, SELinux tweaks, camera/VNDK compatibility). Must have a folder under `ObsidianUI/Devices/`. |
| **TARGET_DEVICE** | The device model whose *firmware* you are actually rebuilding. |
| **FIRMWARE_LINK** | A direct download URL for the target device's firmware (e.g. from samfw.com), used by the CI workflow. |
| **OUTPUT_FILESYSTEM** | `erofs`, `ext4`, or `f2fs` — the filesystem used when repacking the final partition images. |

**Where the Obsidian UI name shows up on-device:** the pipeline sets
`ro.build.display.id` on both `system` and `product` partitions to
`<original build id> V-<version>: Built with Obsidian UI`. This string is
what appears under **Settings → About phone → Software information → Build
number** on the finished ROM.

---

## Repository layout

```
ObsidianUI-fifteen/
├── fifteen.sh                 # main entrypoint / orchestrator
├── release.sh                 # packages output, uploads, creates GitHub release
├── upload.sh                  # GoFile upload helper
├── scripts/
│   ├── ObsidianRom.sh         # core engine: extraction, patching, SELinux, image build
│   ├── debloat.sh             # categorized app removal lists (general, carrier, Samsung, AI, etc.)
│   ├── install_packages.sh    # dependency installer (Ubuntu/Debian, Arch, Fedora)
│   └── extract_img.sh         # generic mount-based image extractor w/ fs_config + file_contexts generation
├── bin/                       # vendored native tools: apktool, omc-decoder, erofs-utils, lpunpack/lpmake, make_ext4fs, f2fs-tools, py_scripts
├── ObsidianUI/
│   ├── Devices/<MODEL>/       # per-stock-device config + floating_feature.xml + Stock/ overlay + extra/
│   ├── vndks/                 # donor VNDK payloads by SDK version
│   ├── Mods/Apps/             # app overlays (JDM-special camera, SDHMS alt, etc.)
│   ├── usefull_things/        # passthrough extras copied straight into OUT/ (Magisk module, DynaPatch, etc.)
│   └── logo/                  # branding assets
└── .github/workflows/fifteen.yml
```

---

## Requirements

Run this on Linux (CI uses `ubuntu-latest`). Install dependencies with:

```bash
bash scripts/install_packages.sh
```

which covers Ubuntu/Debian, Arch, and Fedora. Manually, you'll need roughly:

```
p7zip-full lz4 liblz4-1 liblz4-dev libzstd1 libzstd-dev build-essential
android-sdk-libsparse-utils f2fs-tools fuse2fs fuse e2fsprogs
python3 python3-pip zipalign unzip openjdk-21-jdk jq perl xxd file kmod erofs-utils
```

Most steps require **root** (loop/fuse mounts, `chown`), so local runs use
`sudo bash fifteen.sh ...`.

---

## Usage

### Option A — GitHub Actions (recommended)

1. Go to **Actions → Obsidian UI Tools OS-15 → Run workflow**.
2. Fill in:
   - `STOCK_DEVICE` — a device folder under `ObsidianUI/Devices/` (e.g. `SM-A135F`), or `None`.
   - `TARGET_DEVICE` — the model whose firmware you're rebuilding.
   - `TARGET_DEVICE_CSC` / `TARGET_DEVICE_IMEI` — required arguments for `fifteen.sh`, not used for downloading (the direct link handles that).
   - `FIRMWARE_LINK` — a direct firmware download URL (samfw.com or a Google Drive share link).
   - `OUTPUT_FILESYSTEM` — `erofs` or `ext4`.
   - `COMPRESS_IMG_TO_XZ` — optional, shrinks the release download.
3. On success, the workflow uploads the output zip to GoFile and creates a GitHub Release with the download link.

### Option B — Local run

```bash
git clone <this repo>
cd ObsidianUI-fifteen

bash scripts/install_packages.sh
mkdir -p FW WORK OUT

# place/extract firmware under FW/<TARGET_DEVICE>/ first (see EXTRACT_FIRMWARE /
# EXTRACT_SUPER_IMG in scripts/ObsidianRom.sh for the exact expected layout)

sudo bash fifteen.sh <STOCK_DEVICE> <TARGET_DEVICE> <TARGET_DEVICE_CSC> <TARGET_DEVICE_IMEI> <OUTPUT_FILESYSTEM>
```

Output images land in `OUT/`.

---

## Adding a new STOCK_DEVICE

1. Create `ObsidianUI/Devices/<MODEL>/`.
2. Add a `config` file:
   ```
   STOCK_VNDK_VERSION=33
   STOCK_HAS_AB_SLOT=FALSE
   USE_ALT_SDHMS_APP=TRUE
   STOCK_DEVICE_CHIPSET=Exynos
   STOCK_HAS_SEPARATE_SYSTEM_EXT=FALSE
   STOCK_SIOP_POLICY_FILENAME=siop_a13_exynos850
   STOCK_DVFS_FILENAME=dvfs_policy_exynos850_fhd_xx
   STOCK_DEVICE_CPU_ABILIST=arm64-v8a,armeabi-v7a,armeabi
   STOCK_HAS_ESIM_SUPPORT=FALSE
   ```
3. Add the device's `floating_feature.xml` (from its stock firmware, at
   `system/system/etc/floating_feature.xml`).
4. Optionally add a `Stock/` folder — copied verbatim onto the target
   firmware root (see `APPLY_STOCK_CONFIG` in `scripts/ObsidianRom.sh`).
5. Optionally add an `extra/` folder — copied straight into `OUT/`.
6. `APPLY_STOCK_CONFIG` will refuse to continue if
   `STOCK_DEVICE_CPU_ABILIST` doesn't match the target firmware's ABI list —
   make sure the two are CPU-architecture compatible.

---

## Customizing behavior

- **Debloat lists** — edit the categorized arrays (`DEBLOAT_APPS`,
  `CARRIER_APPS`, `SAMSUNG_APPS`, `SAMSUNG_AI`, ...) in `scripts/debloat.sh`.
- **SELinux patches** — edit `UNSUPPORTED_SELINUX` and the `REMOVE_LINE`
  calls inside `PATCH_SELINUX` in `scripts/ObsidianRom.sh`.
- **Floating features** — edit `APPLY_CUSTOM_FLOATING_FEATURE` (custom,
  applied to every build) or `APPLY_STOCK_ROM_FLOATING_FEATURE` (pulled
  1:1 from the stock device's own floating feature file) in
  `scripts/ObsidianRom.sh`.
- **build.prop tweaks** — edit the `BUILD_PROP` calls inside
  `APPLY_CUSTOM_FEATURES`.
- **Build/branding string** — edit the `ro.build.display.id` value near the
  bottom of `fifteen.sh` if you want different on-device branding text.
- **Framework smali patches** — `PATCH_SSRM`, `PATCH_FLAG_SECURE`,
  `PATCH_SECURE_FOLDER`, `PATCH_PRIVATE_SHARE`, `PATCH_KNOX_GUARD`, and
  `DISABLE_SIGNATURE_VERIFICATION` in `scripts/ObsidianRom.sh` use
  `REPLACE_SMALI_METHOD` to patch decompiled framework jars — extend these
  for further security/permission unlocks.

---

## Known limitations

- `fifteen.sh` decompiles/recompiles `services.jar`, `ssrm.jar`, and
  `samsungkeystoreutils.jar` with apktool on every run — this is the
  slowest stage of the pipeline.
- `ADD_SAMSUNG_FLAGSHIP_APPS` and `FIX_CAMERA` download supplementary app
  packages from an external GitHub release
  (`SN-Abdullah-Al-Noman/Samsung_Special`) at build time — a build will fail
  or partially skip these steps without internet access.
- `PATCH_BT_LIB` relies on a fixed table of hex patch patterns; firmware
  using a `libbluetooth_jni.so` build not covered by the table will skip
  Bluetooth patching.
- Only one `STOCK_DEVICE` profile (`SM-A135F`) ships today.

---

## License

Licensed under the **GNU General Public License v3.0** — see
[`LICENSE`](LICENSE) for the full text.
