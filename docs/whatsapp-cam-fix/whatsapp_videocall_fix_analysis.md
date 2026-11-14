# Analysis: WhatsApp video-call fix (veux) — portability to `~/android/veux/device_xiaomi_veux`

Date: 2026-11-13
Source tree analysed: `/home/balaji/android/device_xiaomi_veux` (acerhizmq, Android 17, branch `17`)
Target tree: `/home/balaji/android/veux/device_xiaomi_veux` (spidey-t, branch `sixteen-qpr2`, PixelOS-style)
ROM frameworks/av checked: mistos, lunaris, infinity, axion (all Android 16)

---

## 1. TL;DR

- The string **"update-sha1sums.py" is NOT the fix.** That script only re-hashes "kanged" (vendor) blobs in `proprietary-files.txt`. It is unrelated to WhatsApp video calls.
- The actual WhatsApp video-call fix lives in **`patches/whatsapp/`** (3 framework patches for `frameworks/av`) plus a **device-tree soong_config line**, **sepolicy**, and **vendor-blob binary patching** in `extract-files.py`.
- The fix is **not** purely framework-side. The framework patch is inert unless the device tree sets `camera.package_name`, and the vendor/sepolicy side is needed for the HAL to write the client package name to a labelled property without SELinux denials.
- It **can be ported** to `~/android/veux/device_xiaomi_veux`: all required vendor blobs are present and contain the target string, the sepolicy type (`vendor_persist_camera_prop`) is provided by the included QCOM sepolicy, and patch 0003 applies cleanly to every ROM's `frameworks/av` tested.
- Caveat: the source fix is written for Android 17 / CamX; the target builds Android 16 ROMs. Patch 0003 applied cleanly in dry-run on all four ROMs, but a real build is still required to confirm.

---

## 2. What the fix actually is

### 2.1 Framework patches (`patches/whatsapp/`)

| Patch | File | What it does |
|---|---|---|
| 0001 | `libcameraservice: Add support to set vendor tag with client package name` | Adds `CAMERA_PACKAGE_NAME` soong-config hook: writes the connecting app's package name into camera session params as a vendor tag. |
| 0002 | `libcameraservice: Fixup! ...` | Escapes the macro value; adds the cflag to `cameraserver`. |
| 0003 | `libcameraservice: Send the client package to the HAL without reconfiguring` | **The actual veux/WhatsApp fix.** See below. |

Patch 0003 (the one that matters) changes:

- `Camera2ClientBase.cpp`: tags each `Camera3Device` with the package of the client that opened it (`setClientPackageName`), instead of relying on a process-wide current package that later (even rejected) connects overwrite.
- `Camera3Device.cpp/h`: keeps `mClientPackageName`; in `configureStreamsLocked` writes the tag into a **copy** (`halSessionParams`) handed to the HAL, **not** into `sessionParams` itself. Previously the `const_cast` write made every capture request look like a session-param change, so CamX rebuilt the session every few hundred ms → WhatsApp got no frames, stock camera got one frame per session.
- Resolves the vendor tag via the device's own `mVendorTagId` instead of the incoming metadata's vendor id.
- `SessionConfigurationUtils.cpp`: adds `persist.vendor.cam.3pjfif` (default off). When set, a non-privileged client's YUV stream ≤ 1080p with JFIF dataspace is configured as `UNKNOWN`, so the Xiaomi HAL does not treat a video call's small YUV stream as a ZSL snapshot.

### 2.2 Device tree (`device.mk`)

```make
$(call soong_config_set_bool,camera,override_format_from_reserved,true)
$(call soong_config_set,libcameraservice,ext_lib,libcameraservice_extension.veux)
# Pass every camera client's package name to the Xiaomi HAL as a session parameter.
$(call soong_config_set,camera,package_name,com.xiaomi.sessionparams.clientName)
```

The `camera.package_name` soong variable is what flips `#ifdef CAMERA_PACKAGE_NAME` in the framework patches. Without it, the entire fix is compiled out (inert).

### 2.3 Vendor blob binary patching (`extract-files.py`)

```python
'vendor/lib64/hw/com.qti.chi.override.so': blob_fixup()
    .binary_regex_replace(b'persist.vendor.camera.clientname',
                          b'persist.vendor.camera.pkgname\x00\x00\x00'),
'vendor/lib64/hw/camera.qcom.so': blob_fixup() ... (same)
'vendor/lib64/com.qti.feature2.rt.so': blob_fixup() ... (same)
```

The HAL originally writes the client name to `persist.vendor.camera.clientname`. The blob fixup renames that property string to `persist.vendor.camera.pkgname`, because the device's sepolicy only labels `pkgname` (see §2.4). Without this, CamX would write to an unlabelled property → SELinux denials.

All three blobs exist in the target vendor tree and each contains the source string `persist.vendor.camera.clientname` (verified), so the fixup is applicable.

### 2.4 SELinux (`sepolicy/vendor/`)

- `property_contexts`: labels `persist.vendor.camera.pkgname` and `persist.vendor.cam.3pjfif` as `u:object_r:vendor_persist_camera_prop:s0`.
- `hal_camera_default.te`: `get_prop`/`set_prop` for `vendor_persist_camera_prop` (CamX writes pkgname from the clientName session param); also reads `build_prop`.
- `vendor_init.te`: `set_prop(vendor_init, vendor_persist_camera_prop)`.
- `cameraserver.te`: `get_prop(cameraserver, vendor_persist_camera_prop)` (for the 3pjfif switch).
- `private/shell.te`: `get_prop(shell, vendor_persist_camera_prop)` (debugging).

`vendor_persist_camera_prop` is defined by the included QCOM sepolicy (`device/qcom/sepolicy_vndr`, `system_restricted_prop`), so no new type declaration is needed in the device tree.

### 2.5 Apply mechanism (`tools/apply-patches.sh` + `vendorsetup.sh`)

- `apply-patches.sh` detects whether each patch is already present in `frameworks/av` (via `whatsapp_has`) and applies only missing ones with `git am -3`.
- `vendorsetup.sh` runs `apply-patches.sh --quiet` during env setup.
- The ROMs tested already contain patches 0001 & 0002 (stock client-package hook); only patch 0003 is missing, so only 0003 actually needs applying.

---

## 3. Dependency on device / vendor trees?

| Component | Depends on device tree? | Depends on vendor tree? | Notes |
|---|---|---|---|
| Framework patch 0003 | Partially — needs `camera.package_name` soong var set in `device.mk` to activate (`#ifdef CAMERA_PACKAGE_NAME`). Code itself is device-agnostic. | No | The patch compiles without the device line, but is inert. |
| `device.mk` soong line | Yes | No | One line. |
| Blob fixup (clientname→pkgname) | No (lives in `extract-files.py`) | Yes — requires the three blobs to be present and contain the source string. | Verified present in target vendor tree. Re-extraction or manual binary patch needed on already-extracted trees. |
| Sepolicy additions | Yes | Indirectly — the property is written by CamX (vendor HAL). | Type provided by QCOM sepolicy (already included). |
| `persist.vendor.cam.3pjfif` switch | No | No (default off) | Optional test switch. |

So: **the fix is a cross-layer change (framework + device + vendor-extraction + sepolicy), not framework-only.** But it does not depend on the *source* dev's specific device-tree code beyond the soong line; the heavy lifting is the framework patch, which is portable.

---

## 4. Portability assessment for `~/android/veux/device_xiaomi_veux`

- Target tree has **no** `patches/`, `tools/apply-patches.sh`, or `camera/` extension dir. So the apply mechanism and torch extension are absent — but those are not required for the WhatsApp fix (torch is a separate component).
- Target `device.mk` already sets `camera.override_format_from_reserved` but **not** `camera.package_name`. → add the line.
- Target vendor tree has all three blobs with the source string. → blob fixup applicable.
- Target BoardConfig includes `device/qcom/sepolicy_vndr/SEPolicy.mk` → `vendor_persist_camera_prop` type available.
- Target `sepolicy/vendor/` has `property_contexts`, `hal_camera_default.te`, `vendor_init.te` but **no** `cameraserver.te`, and none of the camera-persist entries. → add entries.
- `extract-files.py` target has `blob_fixups` dict but lacks the three clientname→pkgname entries. → add them.
- Patch 0003 dry-run succeeded on mistos, lunaris, infinity, axion `frameworks/av` (Android 16). Target builds Android 16 ROMs, so expected to apply, but must be verified against the actual ROM used.

---

## 5. Minimal port checklist (no changes made in this analysis)

1. **Framework (per ROM build):** apply `patches/whatsapp/0003-libcameraservice-Send-the-client-package-to-the-HAL-.patch` to `frameworks/av`. Confirm patches 0001/0002 are already present (they are in all tested ROMs).
2. **Device tree `device.mk`:** add
   `$(call soong_config_set,camera,package_name,com.xiaomi.sessionparams.clientName)`
   next to the existing `override_format_from_reserved` line.
3. **Device tree `sepolicy/vendor/property_contexts`:** add
   ```
   persist.vendor.camera.pkgname   u:object_r:vendor_persist_camera_prop:s0
   persist.vendor.cam.3pjfif       u:object_r:vendor_persist_camera_prop:s0
   ```
4. **Device tree `sepolicy/vendor/hal_camera_default.te`:** add
   ```
   get_prop(hal_camera_default, vendor_persist_camera_prop)
   set_prop(hal_camera_default, vendor_persist_camera_prop)
   ```
5. **Device tree `sepolicy/vendor/vendor_init.te`:** add
   `set_prop(vendor_init, vendor_persist_camera_prop)`.
6. **Device tree `sepolicy/vendor/cameraserver.te`:** create with
   `get_prop(cameraserver, vendor_persist_camera_prop)`.
7. **Device tree `extract-files.py`:** add the three `binary_regex_replace` blob fixups for `chi.override.so`, `camera.qcom.so`, `feature2.rt.so`.
8. **Already-extracted vendor tree:** either re-extract with the fixup, or manually binary-patch the three blobs (`clientname` → `pkgname\0\0\0`).
9. (Optional, for reproducibility) copy `patches/whatsapp/0003-*.patch`, add `tools/apply-patches.sh` + `vendorsetup.sh` hook, like the source tree.

---

## 6. Risks / caveats

- **Android version skew:** source fix targets Android 17 frameworks/av; target builds Android 16 ROMs. Patch 0003 dry-ran cleanly on mistos/lunaris/infinity/axion, but a full build is needed to confirm compilation.
- **"108MP vs 64MP":** the fix addresses a session-reconfiguration bug triggered by third-party apps (WhatsApp) on CamX. Whether the 64MP variant triggers it depends on HAL behaviour; the change is guarded and harmless if inert, but the 3pjfif switch (default off) is the only 108MP-specific knob and is opt-in.
- **Blob fixup on already-extracted trees:** if `vendor_xiaomi_veux` was extracted without the fixup, the three blobs still contain `clientname` and the HAL will write to an unlabelled property → denials. Re-extract or manually patch.
- **No code changes were made** in this analysis; only this document was produced.

---

## 7. Note on `update-sha1sums.py`

`update-sha1sums.py` is a generic LineageOS helper that writes SHA1 hashes of vendor blobs listed in `proprietary-files.txt` (run with/without `-c` to update/clean). It is **not** part of the WhatsApp fix and does not need to be ported for this issue. Its only relevance is keeping `proprietary-files.txt` hashes consistent after re-extracting blobs with the new fixups.