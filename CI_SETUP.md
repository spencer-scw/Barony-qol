# CI Setup — building modded releases

The `Release Build (QOL mod)` workflow (`.github/workflows/release.yml`) builds the modded `barony`
executable for **Linux + Windows** with FMOD (sound) and Steamworks (multiplayer) enabled, and
attaches both binaries to a GitHub Release.

Because FMOD and the Steamworks SDK are **proprietary** and may not be redistributed in the clear,
they cannot be committed to a public repo. Instead you provide them once as an **encrypted archive**
and give CI the key to decrypt it. This mirrors how upstream Barony's CI works, but under your control.

You only do this setup once.

---

## 1. Download the two SDKs (free accounts)

- **FMOD Engine / Core API, version 2.02.14** — https://www.fmod.com/download (create a free account).
  You want the *FMOD Engine* package; inside it is `api/core/` with `inc/` and `lib/`.
- **Steamworks SDK** — https://partner.steamgames.com/downloads/steamworks_sdk.zip (log in with any
  Steam account and accept the SDK license). Inside it is `sdk/` with `public/` and
  `redistributable_bin/`.

## 2. Assemble the `deps` layout

Create a folder and arrange the SDKs **exactly** like this (the paths matter — the CMake find-modules
look for them):

```
deps/
  fmod-linux/                                       (from the FMOD Linux .tar.gz)
    api/core/inc/fmod.hpp ...
    api/core/lib/x86_64/libfmod.so ...
  fmod-win/                                         (from the FMOD Windows installer)
    api/core/inc/fmod.hpp ...
    api/core/lib/x86_64/fmod_vc.lib                 (copied from the installer's api/core/lib/x64/)
  steamworks_sdk/
    sdk/public/steam/steam_api.h ...
    sdk/redistributable_bin/linux64/libsteam_api.so     (Linux)
    sdk/redistributable_bin/win64/steam_api64.lib       (Windows)
```

Notes:
- FMOD ships **per-platform** packages, so there are two FMOD trees; the workflow points `FMOD_DIR`
  at `fmod-linux` / `fmod-win` per job. The Windows import lib goes in `.../lib/x86_64/` (not `x64/`)
  because that's the only lib path the bundled `FindFMOD.cmake` searches.
- NFD is **not** needed here — CI builds it from source on Linux and installs it via vcpkg on Windows.

## 3. Encrypt the archive

```bash
cd deps && zip -r ../deps.zip . && cd ..
# pick a random 256-bit key and 128-bit IV (hex):
KEY=$(openssl rand -hex 32)
IV=$(openssl rand -hex 16)
openssl aes-256-ctr -in deps.zip -out deps.zip.enc -K "$KEY" -iv "$IV"
echo "KEY=$KEY"
echo "IV=$IV"
```

Keep `KEY` and `IV` — you'll paste them into secrets below. `deps.zip.enc` is safe to host publicly
(it's encrypted).

## 4. Host the encrypted archive

Upload `deps.zip.enc` somewhere CI can `curl` it. Easiest: create a GitHub Release in this fork (e.g.
tag `ci-deps`) and attach `deps.zip.enc` as an asset. Copy its download URL.

## 5. Add repository secrets

In the fork: **Settings → Secrets and variables → Actions → New repository secret**. Add:

| Secret     | Value                                             |
| ---------- | ------------------------------------------------- |
| `DEPS_URL` | Direct download URL of `deps.zip.enc` from step 4 |
| `DEPS_KEY` | The `KEY` hex string from step 3                  |
| `DEPS_IV`  | The `IV` hex string from step 3                   |

## 6. Cut a release

Tag and push:

```bash
git tag v5.0.2-qol.1
git push origin v5.0.2-qol.1
```

The workflow builds both platforms and publishes a Release with `barony` (Linux) and `barony.exe`
(Windows) attached. You can also trigger it manually from the **Actions** tab (**Run workflow**) to
test the build without publishing a release.

---

## What your friends do

Download the binary for their OS from the Release and drop it into their existing **Steam** Barony
install (replace or rename the original `barony` / `barony.exe`), then launch through Steam as normal.
Their install already provides the FMOD + Steam runtime libraries, `steam_appid.txt`, and all game
data, so the release ships only the executable. Everyone playing together must run the **same** modded
build — multiplayer is gated on the version string (`v5.0.2-qol`), so modded and vanilla clients won't
mix.

## Notes / first-run caveats

- The **Windows** job uses vcpkg + the repo's bundled `Find*.cmake` modules; the SDL2/PhysFS find
  modules search `vcpkg`'s install tree via the toolchain file. If the first Windows run fails to
  locate a library, the fix is usually pointing the relevant `*_DIR`/`*DIR` env var (or
  `BARONY_WIN32_LIBRARIES`) at the vcpkg `installed/x64-windows` tree — see `INSTALL.md`.
- If you keep the repo **private**, you can skip the encryption and instead fetch the SDK bundle with
  `gh release download` using the built-in `GITHUB_TOKEN`; the encrypted-archive route above is what
  lets the repo stay **public** without redistributing the SDKs in the clear.
