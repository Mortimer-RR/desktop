# Fork notes: upload-integrity fork of the OpenCloud desktop client

This fork carries client-side fixes for upload-integrity gaps until upstream ships
equivalent changes. The server-side fixes live in the reva fork (see its FORK_NOTES.md).

- Upstream: https://github.com/opencloud-eu/desktop
- Base: `f5386cc98` (upstream `main` at the time of the audit)
- Branch layout:
  - `fix/*`: one branch per fix, based directly on the base commit, free of fork-only
    files, so it can be sent upstream as-is.
  - `integrity`: base + this file + all fixes. Build releases from here.
- Tag fork releases `<upstream version>-integrity.N`.
- Clients must be rebuilt and redistributed to staff after every change here.

## Building and testing

Upstream CI builds with KDE Craft (pinned in `.craft.shelf`: Qt 6.11.1, ECM 6.29.0,
qtkeychain 0.16.0, libre-graph-api-cpp-qt-client 1.0.7). For development and tests the
fork was built in an Arch Linux container instead:

| Component | Version used |
| --- | --- |
| Qt | 6.11.2 |
| CMake | 4.4.3 |
| extra-cmake-modules | 6.30.0 |
| qtkeychain-qt6 | 0.17.0 |
| KDSingleApplication | 1.2.1 |
| libre-graph-api-cpp-qt-client | v1.0.7 (built from source) |
| GCC | 16.2.1 |

```dockerfile
FROM archlinux:latest
RUN pacman -Syu --noconfirm --needed base-devel git cmake ninja extra-cmake-modules \
      qt6-base qt6-declarative qt6-tools qt6-svg qt6-imageformats qt6-translations qt6-shadertools \
      qtkeychain-qt6 kdsingleapplication vulkan-headers sqlite zlib inotify-tools ccache
RUN git clone --depth 1 --branch v1.0.7 https://github.com/opencloud-eu/libre-graph-api-cpp-qt-client.git /tmp/lg \
 && cmake -S /tmp/lg/client -B /tmp/lg/build -G Ninja -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_BUILD_TYPE=Release \
 && cmake --build /tmp/lg/build && cmake --install /tmp/lg/build
```

```sh
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug -DBUILD_TESTING=ON
ninja -C build && (cd build && ctest --output-on-failure)
```

All 35 tests pass on the `integrity` branch with this setup. Release builds for staff
(Windows/macOS installers, AppImage) should still go through Craft, as upstream does.

## Divergences from upstream

### 1. Recover TUS uploads whose resume URL is no longer valid

Branch `fix/tus-stale-resume`. Server half: reva fork, fix 4 (460 for checksum
mismatches).

- `src/libsync/propagateuploadtus.cpp` `slotChunkFinished`:
  - If a HEAD to the upload URL fails with **404 or 410** (the server discarded or
    expired the upload) or **403** (the transfer token embedded in the URL expired, and
    the data gateway rejects it), clear the saved resume info, reset `_location` and
    `_currentOffset`, and start a new creation-with-upload in the same run. This happens
    at most once per propagation job (`_restartedUpload`), so a genuine 403 can't loop:
    the new upload's own error is reported normally.
  - On **460 Checksum Mismatch**, clear the saved resume info before the normal error
    handling, so the next attempt starts a new upload.
- `classifyError` is unchanged: 460 falls through to `NormalError` (retried).
- Tests: `test/testtusresume.cpp` with a small fake tus server (creation-with-upload
  POST, PATCH, HEAD on top of the fake remote): resume from the server offset still
  works (HEAD, PATCH, PATCH); stale resume info answered with 404/410/403 → the file
  uploads in the same sync (HEAD, POST, PATCH, PATCH) and the resume info is gone; 460 on
  the last PATCH → resume info cleared, next sync starts with POST.
- Before the fix the stale-resume cases failed the sync (and would on every later sync)
  and the 460 case kept the resume info.

Upstream issue draft:

> **TUS: uploads get stuck when the saved resume URL is no longer valid**
>
> The resume info is only cleared after a successful upload. If the server discards an
> upload (e.g. after a checksum mismatch, or because it expired), the next sync sends
> HEAD to the old URL, gets 404, and `slotChunkFinished` passes it to
> `commonErrorHandling` without clearing the resume info. The file never syncs again
> until it is modified locally. The same happens with 403 once the transfer token in
> the upload URL has expired.

## Notes

- The handoff named only 404/410. The owner approved adding 403 (expired transfer
  token) on 2026-10-01.
