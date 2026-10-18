English | [简体中文](README.md)

# xtby · Android kernel / driver manager

An Android app written in **Kotlin + Jetpack Compose**: it fetches a "kernel file" on device, downloads and filters "driver scripts", flashes a driver into `/data/han/drivers` through `su`, and embeds a terminal that can run the kernel.

The UI uses a "liquid glass" style with three bottom tabs:

| Tab | Screen | Purpose |
| --- | --- | --- |
| Launch | `LaunchScreen` | Kernel status, fetch kernel, run kernel, **the only screen with a terminal** |
| Driver | `DriverScreen` | Download / import driver packages, filter by current kernel, flash, inspect file contents |
| Settings | `SettingsScreen` | Kernel auto-input steps, device info, root status, about page |

> This repository contains **no kernel or driver files**. Everything executable is fetched at runtime from the configured download URLs, or picked locally by the user.

---

## 1. Project layout

```
.
├── settings.gradle.kts          # rootProject.name = "xtby", include(":app")
├── build.gradle.kts             # top-level build script
├── gradle/libs.versions.toml    # dependency version catalog (AGP / Kotlin / Compose BOM)
├── README.md                    # 简体中文
├── README.en.md                 # English (this file)
└── app/
    ├── build.gradle.kts         # applicationId, SDK levels, Compose, optional CMake/NDK
    └── src/main/
        ├── AndroidManifest.xml  # INTERNET permission only, single Activity
        ├── cpp/                 # optional native PTY (forkpty); the app still works without it
        │   ├── CMakeLists.txt
        │   └── pty_terminal.cpp
        ├── res/
        │   ├── values/strings.xml               # app_name = xtby
        │   └── xml/network_security_config.xml  # cleartext HTTP allow-list
        └── java/com/example/myapplication/
            ├── MainActivity.kt    # entry point + all screens + kernel flow + su exec (largest file)
            ├── Driver.kt          # driver package definitions, download/unzip, cache, version matching, content reader
            ├── KernelBinary.kt    # file type detection (ELF / script / zip / self-extracting gzip payload)
            ├── KernelAutoInput.kt # state machine for auto-answering kernel menus
            ├── PtyTerminal.kt     # terminal backend: native PTY, falls back to a pipe shell
            ├── LiquidGlass.kt     # glass-style components and the bottom tab bar
            └── ui/theme/          # theme and colors
```

---

## 2. How it builds

### 2.1 Build setup

A plain Android Gradle project with no private build scripts and no prebuilt artifacts:

- Plugins: `com.android.application` + `org.jetbrains.kotlin.plugin.compose`
- `compileSdk = 36` (minor 1), `minSdk = 24`, `targetSdk = 36`
- Java/Kotlin target `VERSION_11`
- Compose BOM `2026.02.01`, Material 3
- Very few dependencies: `activity-compose`, `core-ktx`, `lifecycle-runtime-ktx`, Compose (ui / graphics / tooling / material3)

Build commands:

```bash
./gradlew :app:assembleDebug     # debug APK
./gradlew :app:assembleRelease   # release APK (no signingConfig yet, add your own)
```

### 2.2 The native PTY is optional

`app/src/main/cpp/` contains a minimal `forkpty` implementation. Once `libptyterminal.so` is compiled, the terminal gets a real tty (interactive menus, `tput`, cursor control behave correctly).

But **the .so is not required**:

- If the build environment has no NDK / no native build task, the APK simply does not contain `libptyterminal.so`;
- `PtyTerminal.start()` first tries `System.loadLibrary`; if the library is missing it **does not throw** — it automatically degrades to a **pipe shell** via `ProcessBuilder("/system/bin/sh", "-c", ...)` and prints one warning line in the terminal;
- The fallback has no tty, so menu-style scripts may not react, but output is visible, input works, and scripts do run.

So **you can build and use the app without an NDK**; only terminal fidelity is reduced. For full terminal support you need `ndkVersion = "27.3.13750724"` + CMake (already configured, `abiFilters arm64-v8a`).

### 2.3 Two layers of state

- `MainScaffold` is the **single owner of shared state** (terminal lines, command, device info, root status, download progress, kernel steps, driver state, ...); the three screens only receive parameters and callbacks;
- Mutable driver state lives in `DriverSlot` / `DriverState` (Compose `mutableStateOf` / `mutableStateMapOf`);
- Kernel input steps are persisted in `SharedPreferences("tang_preferences")` under `kernel_steps` (serialized line by line as `"{maxOption}\t{value}"`, max 12 steps).

---

## 3. How it runs

### 3.1 Kernel: fetch → write to disk → execute

```
cloud download or local pick
      │
      ▼
resolveKernelScript()        type is decided by **file header**; a zip is unpacked, preferring ELF, then .sh
      │
      ▼
prepareKernelScript()        extracts the gzip payload of the self-extracting wrapper, then su writes:
                             /data/han/cnhk.sh
                             /data/han/cnhk.sh.payload (when a payload exists, chmod 777)
      │
      ▼
kernelRunFragment()          device-side shell:
                             ① run ./cnhk.sh.payload first (if it is an ELF)
                             ② else ./cnhk.sh (if it is an ELF, execute directly)
                             ③ else sh ./cnhk.sh (script)
```

Key points:

- **The extension is ignored.** Kernels from file hosts often have no suffix, and an ELF may be named `cnhk.sh`, so `KernelBinary.kt` only reads the header: `0x7F ELF` / `PK` / `#!` / plain text;
- **An ELF must never be fed to `sh`** — it produces a wall of syntax errors; scripts with a shebang run either directly or via `sh`;
- The **self-extracting wrapper** (`tail -n +44 | gzip -cd`) **silently exits 256** when the device lacks `gzip` / `mktemp` or `TMPDIR` is not writable. The app therefore extracts the payload itself with `GZIPInputStream` and runs the payload directly;
- Kernels dropped into `/data/han` by hand are also detected: `kernelPickFragment()` prefers `cnhk.sh`, otherwise takes the first kernel-looking file in that directory (skipping `*.installed`, `*.tmp`, `*.payload`, `PK`-prefixed archives and the `drivers` directory), and `chmod 777` is applied before running.

### 3.2 Drivers: download → unzip → filter → flash

```
download zip (or pick a local zip)
      │
      ▼  extractDriverArchive()  recursive unzip with zip-slip protection
filesDir/drivers/<id>/           and writes a .source marker (source URL / local:<zip name>)
      │
      ▼  detectKernelInfo() reads os.version (no root needed), parsed to "6.6"
      │  filterByKernel()   shows only files matching major.minor, switchable to "show all"
      ▼
flashDriverScript()              su executes:
                                 mkdir -p /data/han/drivers
                                 cp file → chmod 777 → run (ELF directly / script via sh)
                                 date > <file name>.installed   ← flash marker
```

- Each driver package maps to one `DriverPackage` and gets its own cache directory, so it is **downloaded/unzipped only once**;
- `.source` stores the source URL, so **changing the link (domain or id) invalidates the old cache** automatically instead of showing stale contents;
- Flash time is recorded in `/data/han/drivers/<name>.installed`, so no script is re-executed just to detect state;
- The "view full content" button uses `readDriverContent()` line by line: ANSI escapes are stripped and binary lines are replaced with a placeholder, so "script + embedded binary" packages still show their readable part (limit 400 KB).

### 3.3 Terminal and auto-input

- `PtyTerminal`: native PTY (`forkpty`) first, pipe shell as fallback; identical interface (`write` / `resize` / `close` / `isRunning`);
- `KernelAutoInput`: 4000-character sliding window; recognizes numbered menus at the start of a line and prompts like "select / enter / choose", then writes the user-configured steps in order;
- Step contents are user-defined (`KernelInputStep(value, maxOption)`); `maxOption` is only used to bound prompt detection; **the first step with an empty value, and every step after it, is not auto-filled**;
- After answering, `window.clear()` runs so the previous prompt cannot trigger the next step twice.

### 3.4 Privacy handling

The terminal and driver cards are both "customer-visible". If the download source is your own private domain, it should not appear there:

- `hidePrivateLinks()` replaces any whole URL containing one of `PrivateDownloadHosts`, plus bare domains without `http://`, with `（下载地址已隐藏）`;
- Only your own domains are masked; other sites such as `https://example.com/a?b=1` are left untouched;
- Every terminal line goes through `appendLine()`, which applies this filter, and driver card errors use the same function.

---

## 4. Changing download links / servers (the most common edit)

> No download source yet? Jump to **[section 5](#5-where-to-host-downloads-self-hosted-cloud-drive-recommended-r2-cloud-drive)** and self-host R2-Cloud-Drive — the direct links it produces match this project exactly.

### 4.1 Kernel URL

File: `app/src/main/java/com/example/myapplication/MainActivity.kt` (top of file, around line 110)

```kotlin
// Kernel download URL
private const val KernelUrl =
    "https://域名/api/share-download?id=7c71a3d0cde94a10ab57c14834babff9"
```

Replace it with any address that **returns the file body directly**, e.g.:

```kotlin
private const val KernelUrl = "https://example.com/kernel/cnhk.sh"
// or your own share endpoint
private const val KernelUrl = "https://example.com/api/share-download?id=xxxx"
```

Requirements and caveats:

- Must return 200 with the file body (script / ELF / suffix-less zip are all supported, detected by header);
- HTTPS is the easy path; if you must use **cleartext HTTP**, add it to the allow-list in `app/src/main/res/xml/network_security_config.xml`, otherwise Android blocks the request:

  ```xml
  <domain-config cleartextTrafficPermitted="true">
      <domain includeSubdomains="false">your-domain-or-ip</domain>
  </domain-config>
  ```

- Share endpoints generate on the fly, so the first byte can be slow; `readTimeout` in `downloadKernelScript()` is **60 s** — do not shrink it casually;
- If the server returns a zip, `resolveKernelScript()` unpacks it and prefers the ELF inside.

### 4.2 Driver package list

File: `app/src/main/java/com/example/myapplication/Driver.kt` (around lines 44-65)

```kotlin
private const val DriverShareHost = "https://域名/api/share-download?id="

val DriverPackages = listOf(
    DriverPackage(
        id = "RTDEV",
        name = "RTDEV",
        caption = "RT driver · 16 kernel scripts",
        url = DriverShareHost + "a1ee4f083d6d4166b7f0ceaa3f2dae30"
    ),
    // ...
)
```

- `id` doubles as the **cache directory name** (`filesDir/drivers/<id>`), so it must be unique per package; changing `id` means a fresh cache;
- `url` can be any direct zip link — the share endpoint is not required;
- Package layout does not matter: subdirectories or flat root both work; only `.sh` / `.txt` / ELF files are collected;
- To add a package, append one entry to `listOf(...)` and the UI grows another driver card automatically;
- Changing the URL invalidates the old cache (the `.source` marker is compared against the URL), so no manual data clearing is needed.

### 4.3 Private domain mask (must update after changing domain)

If the new link still uses a private domain that should not leak into the terminal or UI:

File: `MainActivity.kt`, just above `hidePrivateLinks()`

```kotlin
private val PrivateDownloadHosts = listOf("915161.xyz")
```

Replace with your domain(s):

```kotlin
private val PrivateDownloadHosts = listOf("example.com", "cdn.example.com")
```

If you intend to make the source fully public, you can also drop this layer entirely (make `hidePrivateLinks` return `text` unchanged) and the "（download address hidden）" placeholder disappears.

### 4.4 App name, package name, icon

| What you want to change | Where |
| --- | --- |
| Launcher label | `app/src/main/res/values/strings.xml` → `app_name` (currently `xtby`) |
| Package identity | `app/build.gradle.kts` → `applicationId` (currently `com.tencent.android.qqdownloader`) |
| Kotlin package / namespace | `app/build.gradle.kts` → `namespace`, plus source directories and `package` declarations |
| Version | `app/build.gradle.kts` → `versionCode` / `versionName` |
| Icon | `res/mipmap-*`, `res/drawable/ic_launcher_*` |
| Root project name | `settings.gradle.kts` → `rootProject.name` |

> For a self-hosted open-source build, changing `applicationId` and `namespace` to your own (e.g. `com.yourname.xtby`) is strongly recommended; otherwise it collides with other builds and cannot be signed for distribution.

### 4.5 Contact info / about page

`AboutScreen()` in `MainActivity.kt` (around line 2127): the Telegram group `https://t.me/BYYXnb` and QQ group `427975011` are hardcoded strings — replace or remove them before publishing.

### 4.6 Runtime path constants (usually untouched)

Top of `MainActivity.kt`:

```kotlin
private const val KernelDirectory        = "/data/han"
private const val KernelFileName         = "cnhk.sh"
private const val DriverRootDirectory    = "/data/han/drivers"
private const val KernelPayloadFileName  = "cnhk.sh.payload"
```

If you change the directory, `kernelPickFragment()` / `kernelRunFragment()` / `flashDriverScript()` all read these constants — no per-call-site edits needed.

---

## 5. Where to host downloads: self-hosted cloud drive (recommended: R2-Cloud-Drive)

The app demands almost nothing from a download source: **an HTTPS URL that returns the file body on a plain GET**. Object storage, static hosting, or a self-hosted drive all work; no private API is involved.

A free, self-deployable option:

> **[HandsomeMJZ/R2-Cloud-Drive](https://github.com/HandsomeMJZ/R2-Cloud-Drive)** — a cloud drive built on Cloudflare Workers + R2, Material Design, single `worker.js`, no build step.

Why it fits: its **share direct-link format is exactly the one this project already uses**:

```
https://your-domain/api/share-download?id=share-suffix
```

So once it is deployed, you only replace `KernelUrl` and `DriverPackages[].url` with links under your own domain — **no other code changes**.

### 5.1 Relevant features

- Upload / download / rename / new folder, grid and list views, dark mode;
- **Custom share links**: `/s/<suffix>`, optionally with password, expiry, and visit limits;
- **Password-free single-file shares expose a "copy direct link"**: `/api/share-download?id=my-file` without opening the share page first → exactly what `HttpURLConnection` needs;
- `/shared` directory: public read-only, good for public packages; the direct link is `/api/download?path=shared%2F<file>` (`path` must be URL-encoded);
- Chunked upload / Range download, multi-account R2 nodes pooling capacity;
- Optional `ACCESS_PASSWORD` for the web UI; `SITE_TITLE`, `CLOUD_ICON_URL`, `LOGIN_BACKGROUND_URL` for branding.

### 5.2 Deployment (condensed from the official docs)

1. Create an **R2 bucket** in Cloudflare (note: the Cloudflare account must have PayPal bound to create R2);
2. Create a **D1 database** under **Workers & Pages → D1**;
3. Create a **Worker**, paste the whole `worker.js` into the online editor, Deploy (no build; upgrades are the same paste-and-Deploy);
4. Under **Settings → Bindings** add two bindings; names must match exactly:
   - `R2_BUCKET` → your R2 bucket
   - `DB` → your D1 database (tables are created automatically on first request)
5. (Optional) add variables such as `ACCESS_PASSWORD`;
6. You get `https://<project>.<account>.workers.dev`, or bind your own domain via **Triggers → Custom Domains**.

Full details, multi-node setup and the API reference are in that repo's `README.MD`, `API说明文档.MD` and `TECHNICAL_DOC.MD`.

### 5.3 Get a direct link and wire it in

1. Upload the kernel file (script / ELF / suffix-less zip) and the driver zips;
2. Select the file → **Share settings** → set a suffix (1-64 chars of letters, digits, `_`, `-`; must start with a letter or digit; lowercased);
3. **Do not set a password** (the app has no login/password flow). Save, then click **copy direct link**:

   ```
   https://your-domain/api/share-download?id=my-kernel
   ```

4. Put it into the code:

   ```kotlin
   // top of MainActivity.kt
   private const val KernelUrl =
       "https://your-domain/api/share-download?id=my-kernel"
   ```

   ```kotlin
   // Driver.kt
   private const val DriverShareHost = "https://your-domain/api/share-download?id="

   val DriverPackages = listOf(
       DriverPackage(
           id = "MYDRIVER",                 // only used as the local cache directory name
           name = "My driver",
           caption = "Custom driver package",
           url = DriverShareHost + "my-driver"
       )
   )
   ```

### 5.4 Common pitfalls

- **Suffix charset**: letters, digits, `_` and `-` only. Avoid CJK characters or spaces — they are unmaintainable once URL-encoded;
- **Changing the suffix / refreshing the link invalidates the old address.** After that you must update the constants and ship a new build, otherwise older clients get a 404 (this app shows "下载失败：HTTP 404"). Note the link change in your release notes;
- **Use one suffix per driver package**, not a directory share: a directory share's `id` points at a directory, and the client would not receive a zip;
- **Expiry / visit limits**: direct links are still subject to share expiry and visit counts. For app-facing links **do not set a visit limit**, or all users stop downloading once traffic adds up;
- **`ACCESS_PASSWORD` only protects the web UI**; it does not affect direct links, which are still subject to the two limits above;
- **Domain masking**: after changing the domain, update `PrivateDownloadHosts` (see 4.3), otherwise your domain shows up in the terminal;
- **HTTPS is enough**: `workers.dev` and custom domains are HTTPS, so `network_security_config.xml` needs no change; only a cleartext-HTTP self-hosted server requires the allow-list;
- **Licensing**: that repository currently has **no license declared on GitHub** (license field is empty). Self-deploying (running `worker.js` on your own Cloudflare account) is normally fine, but **confirm the author's permission before redistributing or using commercially**, e.g. via an issue or the author's Telegram. Your own project, by contrast, should ship a `LICENSE` (Apache-2.0 / MIT recommended).

### 5.5 Other options

| Option | Good for | Notes |
| --- | --- | --- |
| GitHub Releases direct link | Near-zero cost, small packages | `releases/download/<tag>/<file>` is a plain GET; mind reachability from mainland China |
| R2 public bucket + custom domain | Full control, already on Cloudflare | Make the bucket public; the object URL is the file URL; no admin UI |
| Any object storage (S3-compatible, Qiniu, Upyun, ...) | You already have one | Just generate a public-read URL |
| Self-hosted Nginx / Caddy static directory | You have a VPS | Must configure HTTPS, otherwise add the cleartext allow-list; host only public files |

> Whichever you pick, host **kernels and drivers in separate directories**: kernels change often, driver packages are large, and mixing them makes caching and invalidation harder.

---

## 6. Permissions and prerequisites

- `AndroidManifest.xml` requests only `INTERNET`;
- Every write to `/data/han` and every driver flash goes through **`su -c`**, i.e. **the device must be rooted and must grant this app root**;
- The app requests no storage permission: kernel/driver files download into `cacheDir`, unpack into `filesDir`, and local import uses SAF (`OpenDocument`, MIME `*/*`);
- The device kernel version is read from `System.getProperty("os.version")` (no root), falling back to `/proc/version`.

---

## 7. FAQ

**Q: The terminal says "PTY native library unavailable, switched to pipe shell"**
A: The APK has no `libptyterminal.so` (the native/NDK task did not run at build time, or the device ABI is not arm64-v8a). Everything still works, but without a tty interactive menus may not respond. To fix it, make sure the CMake task runs during the build and keep `abiFilters` in `externalNativeBuild`.

**Q: The download succeeds but the kernel "never runs" and the terminal prints nothing**
A: Most likely the kernel is a self-extracting script and the device lacks `gzip`/`mktemp` or `TMPDIR` is not writable. This project extracts the `.payload` and runs it first; if the `prepareKernelScript()` log does not say the payload was extracted, the file is not a standard `#!` + gzip structure — check what the server actually returns.

**Q: I changed my link but the UI still shows the old drivers**
A: An existing `filesDir/drivers/<id>` cache is not re-downloaded. Changing `url` invalidates it automatically; if only the server changed and the URL did not, tap re-download on the card (or clear app data).

**Q: A cleartext HTTP link never downloads**
A: targetSdk 28+ blocks cleartext traffic by default; edit `network_security_config.xml`. Note `<application android:networkSecurityConfig>` already points at that file.

**Q: `use` / `responseCode` in `downloadKernelScript` shows "Unresolved reference"**
A: That is the custom `HttpURLConnection.use` extension at the end of the file; some analyzers misreport it — it compiles fine.

---

## 8. Pre-release checklist

- [ ] Replace `KernelUrl` and `DriverPackages[].url` (or delete packages you do not own)
- [ ] Confirm the download source is deployed (see section 5: self-hosted drive / object storage / Releases) and document how to change links
- [ ] App-facing share links: **no password, no visit limit**, or all users fail at once
- [ ] Update `PrivateDownloadHosts`, or remove the masking layer
- [ ] Replace `applicationId` / `namespace` / `app_name` / icons
- [ ] Clean up the personal contact info in `AboutScreen`
- [ ] Make sure `network_security_config.xml` has no leftover old IP/domain
- [ ] Add a `LICENSE` (e.g. Apache-2.0 / MIT) and a `signingConfig`
- [ ] If you redistribute kernel/driver files themselves, confirm you are allowed to

---

## 9. Disclaimer

This project is only an execution shell: it bundles, modifies and warrants no kernel or driver. Flashing kernels/drivers, gaining root, and running third-party scripts all risk **bricking the device, data loss and voiding the warranty** — evaluate and accept the risk yourself. Follow the laws of your jurisdiction and the licenses of the software/services involved.
