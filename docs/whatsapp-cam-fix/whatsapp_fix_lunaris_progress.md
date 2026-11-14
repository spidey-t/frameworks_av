# Progress: WhatsApp video-call fix applied to `~/android/lunaris`

Date: 2026-11-13
Source reference: `/home/balaji/android/device_xiaomi_veux` (acerhizmq, branch `17`) — see also https://github.com/acerhizmq
Target: `/home/balaji/android/lunaris` (ROM source; device tree `device/xiaomi/veux` on branch `lunaris`)

## What was applied

### 1. Framework (`frameworks/av`) — commit created
- Verified patches 0001 & 0002 (stock client-package hook) already present:
  - `getCurrPackageName` in `CameraService.cpp` (count 1)
  - `-DCAMERA_PACKAGE_NAME=` in `camera/cameraserver/Android.bp` (count 1)
- Applied patch 0003 only:
  - `git am -3 /home/balaji/android/device_xiaomi_veux/patches/whatsapp/0003-libcameraservice-Send-the-client-package-to-the-HAL-.patch`
  - Result commit: `eee8c55 libcameraservice: Send the client package to the HAL without reconfiguring`
  - Verified `setClientPackageName` now present in `Camera3Device.h`.

### 2. Device tree (`device/xiaomi/veux`) — uncommitted working-tree changes
- `device.mk`: added
  `$(call soong_config_set,camera,package_name,com.xiaomi.sessionparams.clientName)`
  after the existing `override_format_from_reserved` line.
- `sepolicy/vendor/property_contexts`: added
  `persist.vendor.camera.pkgname  u:object_r:vendor_persist_camera_prop:s0`
  `persist.vendor.cam.3pjfif     u:object_r:vendor_persist_camera_prop:s0`
- `sepolicy/vendor/hal_camera_default.te`: added
  `get_prop(hal_camera_default, vendor_persist_camera_prop)` and
  `set_prop(hal_camera_default, vendor_persist_camera_prop)`.
- `sepolicy/vendor/vendor_init.te`: added
  `set_prop(vendor_init, vendor_persist_camera_prop)`.
- `sepolicy/vendor/cameraserver.te`: created with
  `get_prop(cameraserver, vendor_persist_camera_prop)`.
- `extract-files.py`: added 3 `binary_regex_replace` blob fixups
  (`com.qti.chi.override.so`, `camera.qcom.so`, `com.qti.feature2.rt.so`):
  `persist.vendor.camera.clientname` → `persist.vendor.camera.pkgname\0\0\0`.
  Verified Python syntax OK.

### 3. Vendor blobs (`vendor/xiaomi/veux/proprietary/...`) — binary-patched in place
- All three blobs contained `persist.vendor.camera.clientname`; replaced with `persist.vendor.camera.pkgname` + 3 NUL bytes (length-preserving).
- Verified each now contains only `persist.vendor.camera.pkgname`.

## Not applied (intentionally / out of scope)
- Patch apply-mechanism (`tools/apply-patches.sh`, `patches/`, `vendorsetup.sh` hook) — not ported; patch was applied directly to frameworks/av.
- Torch extension (`camera/` dir, `libcameraservice.ext_lib`) — separate component, not part of WhatsApp fix.
- `private/shell.te` get_prop for `vendor_persist_camera_prop` — optional (debugging); skipped.
- `persist.vendor.cam.3pjfif` is left default-off (opt-in test switch).
- No commits made on the device tree (changes left as working-tree edits for review).
- Other ROM trees (mistos, infinity, axion) not touched.

## Verification performed
- `frameworks/av`: `git am` succeeded; `setClientPackageName` present.
- `extract-files.py`: `ast.parse` OK.
- Vendor blobs: grep confirms `pkgname` only, no `clientname`.
- Device tree git status: 5 modified + 1 new file (cameraserver.te).

## Build attempt 1 (failed)
- Command: `bash -ic 'cd ~/android/lunaris && s1 && l2v && l3'` (background).
- Result: **FAILED — "No space left on device"** (disk 100% full, only ~621 MB free on /home).
- Failure points: R8 dex compilation (MediaProvider) and clang backend IO; ninja stopped at 44% (89957/204254).
- Environment/disk-space failure, NOT a compile error in the WhatsApp fix changes.

## Build attempt 2 (succeeded)
- After freeing disk space, reran `bash -ic 'cd ~/android/lunaris && s1 && l2v && l3'` (background).
- Initial SELinux neverallow failure: `hal_camera_default` could not `set` `vendor_persist_camera_prop` because Lunaris defines that type as `system_vendor_config_prop` (only vendor_init may set). Fixed by mapping the two properties to the existing `vendor_camera_prop` type (`vendor_restricted_prop`, writable by vendor domains such as hal_camera_default) and dropping the vendor_init set_prop.
- Result: **BUILD COMPLETED SUCCESSFULLY** (18:53).
- Output: `out/target/product/veux/Lunaris-AOSP-veux-COMMUNITY-3.13-GMS-2026111316.zip` (2.3 GB, user build).
- This confirms the WhatsApp fix (framework patch 0003 + device.mk soong config + sepolicy + vendor blob patch) compiles and links in the Lunaris/veux build.

## Git maintenance (done)
- `device/xiaomi/veux`: 3 commits pushed to `spidey-t/device_xiaomi_veux` (`lunaris` branch): `08b6a7d`, `f516812`, `5b83f60`.
- `frameworks/av`: commit `eee8c55` pushed to `spidey-t/frameworks_av` (`lunaris` branch). Remote URL switched to HTTPS (PAT auth). Local clone unshallowed to make push possible.
- `vendor/xiaomi/veux`: commit `7cb996c` pushed to `spidey-t/vendor_xiaomi_veux` (`lunaris` branch), created from `sixteen-qpr2`, containing the 3 binary-patched camera blobs.

## Rollback
- Framework: `git -C frameworks/av reset --hard` to previous HEAD (backup branch not created; HEAD before was `b7e5fda`). Or `git revert eee8c55`.
- Device tree / vendor blobs: `git checkout -- ...` for tracked files; revert the three vendor blobs from a fresh extract or git; remove `cameraserver.te`.

## Notes
- The framework patch is device-agnostic but is inert unless `camera.package_name` soong var is set (done in device.mk).
- The blob fixup in `extract-files.py` only takes effect on next extraction; already-extracted vendor blobs were patched manually above.
- Source fix targets Android 17; lunaris is Android 16. Patch 0003 dry-ran cleanly on mistos/lunaris/infinity/axion before applying; full build still required to confirm compilation.