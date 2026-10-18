简体中文 | [English](README.en.md)

# xtby · Android 内核 / 驱动管理客户端

一个用 **Kotlin + Jetpack Compose** 写的 Android 应用：在设备上获取「内核文件」、获取并筛选「驱动脚本」、通过 `su` 把驱动刷进 `/data/han/drivers`，并内置一个可以跑内核的终端。

界面走「液态玻璃」风格，底部三个 Tab：

| Tab | 页面 | 作用 |
| --- | --- | --- |
| 启动 | `LaunchScreen` | 内核状态、获取内核、执行内核、**唯一带终端的页面** |
| 驱动 | `DriverScreen` | 下拉/本地导入驱动包、按当前内核过滤、刷入、查看文件内容 |
| 设置 | `SettingsScreen` | 内核自动输入步骤、设备信息、root 状态、关于页 |

> 本仓库**不含任何内核或驱动文件**。所有可执行内容都是运行时从配置的下载链接拉取，或由用户从本地选取。

---

## 一、工程结构

```
.
├── settings.gradle.kts          # rootProject.name = "xtby"，include(":app")
├── build.gradle.kts             # 顶层构建脚本
├── gradle/libs.versions.toml    # 依赖版本清单（AGP / Kotlin / Compose BOM）
├── README.md                    # 简体中文（默认）
├── README.en.md                 # English
└── app/
    ├── build.gradle.kts         # 包名、SDK 版本、Compose、可选 CMake/NDK
    └── src/main/
        ├── AndroidManifest.xml  # 只有 INTERNET 权限，单 Activity
        ├── cpp/                 # 可选的原生 PTY（forkpty），不编译也能用
        │   ├── CMakeLists.txt
        │   └── pty_terminal.cpp
        ├── res/
        │   ├── values/strings.xml            # app_name = xtby
        │   └── xml/network_security_config.xml  # 明文 HTTP 白名单
        └── java/com/example/myapplication/
            ├── MainActivity.kt    # 入口 + 全部界面 + 内核流程 + su 执行（最大文件）
            ├── Driver.kt          # 驱动包定义、下载解包、缓存、版本匹配、内容读取
            ├── KernelBinary.kt    # 文件类型判定（ELF / 脚本 / zip / 自解压 gzip 负载）
            ├── KernelAutoInput.kt # 内核交互菜单的自动输入状态机
            ├── PtyTerminal.kt     # 终端后端：原生 PTY，失败退化为管道 shell
            ├── LiquidGlass.kt     # 玻璃风格组件与底部 TabBar
            └── ui/theme/          # 主题与配色
```

---

## 二、构建原理

### 1. 构建方式

标准 Android Gradle 工程，没有任何私有的构建脚本或预编译产物：

- 插件：`com.android.application` + `org.jetbrains.kotlin.plugin.compose`
- `compileSdk = 36`（minor 1）、`minSdk = 24`、`targetSdk = 36`
- Java/Kotlin 编译目标 `VERSION_11`
- Compose BOM `2026.02.01`，Material 3
- 依赖很少：`activity-compose`、`core-ktx`、`lifecycle-runtime-ktx`、Compose（ui / graphics / tooling / material3）

构建命令：

```bash
./gradlew :app:assembleDebug     # 调试 APK
./gradlew :app:assembleRelease   # 发布 APK（未配置签名，需自行加 signingConfig）
```

### 2. 原生 PTY 是可选项

`app/src/main/cpp/` 里有一个 `forkpty` 的极简实现，编出 `libptyterminal.so` 后终端能拿到真实 tty（交互式菜单、`tput`、光标控制才正常）。

但**这个 .so 不是必需的**：

- 如果构建环境没有 NDK / 没有编译 native 任务，APK 里就不会有 `libptyterminal.so`；
- `PtyTerminal.start()` 会先尝试 `System.loadLibrary`，取不到库时**不抛异常**，而是自动退化成 `ProcessBuilder("/system/bin/sh", "-c", ...)` 的**管道 shell**，并在终端里打印一行警告；
- 退化模式没有 tty，菜单类脚本可能不响应，但输出能看、命令能发、脚本能跑。

所以：**没装 NDK 也能正常编译使用**，只是终端体验降级。想让终端完整工作，需要 `ndkVersion = "27.3.13750724"` + CMake（工程里已配置，`abiFilters arm64-v8a`）。

### 3. 两层状态管理

- `MainScaffold` 是**唯一持有共享状态**的地方（终端行、命令、设备信息、root 状态、下载进度、内核步骤、驱动状态等），三个页面只接收参数和回调；
- 驱动自身的可变状态放在 `DriverSlot` / `DriverState`（Compose `mutableStateOf` / `mutableStateMapOf`）；
- 内核输入步骤持久化在 `SharedPreferences("tang_preferences")` 的 `kernel_steps` 键（按行序列化 `"{上限}\t{内容}"`，上限 12 步）。

---

## 三、运行原理

### 1. 内核：获取 → 落盘 → 执行

```
云端下载 或 本地选择
      │
      ▼
resolveKernelScript()        按**文件头**判断类型；是 zip 就解包，优先挑 ELF，其次 .sh
      │
      ▼
prepareKernelScript()        解出自解压壳里的 gzip 负载，并 su 写入：
                             /data/han/cnhk.sh
                             /data/han/cnhk.sh.payload（有负载时，chmod 777）
      │
      ▼
kernelRunFragment()          设备侧 shell：
                             ① 优先跑 ./cnhk.sh.payload（ELF 时）
                             ② 其次 ./cnhk.sh（本身是 ELF 时，直接执行）
                             ③ 最后 sh ./cnhk.sh（脚本）
```

关键点：

- **不看扩展名**。网盘给的内核经常没后缀，也可能把 ELF 命名成 `cnhk.sh`，所以 `KernelBinary.kt` 只读文件头：`0x7F ELF` / `PK` / `#!` / 纯文本；
- **ELF 绝不能喂给 `sh`**，否则报一堆语法错误；带 shebang 的脚本直接执行或 `sh` 都能跑；
- **自解压壳**（`tail -n +44 | gzip -cd`）在设备缺 `gzip` / `mktemp`、或 `TMPDIR` 不可写时会**静默 exit 256**。所以应用自己用 `GZIPInputStream` 把负载解出来，运行时直接跑负载；
- 用户手动放进 `/data/han` 的内核也能被认出来：`kernelPickFragment()` 会优先 `cnhk.sh`，否则取目录里第一个像内核的文件（跳过 `*.installed`、`*.tmp`、`*.payload`、`PK` 开头的压缩包和 `drivers` 目录），运行前统一 `chmod 777`。

### 2. 驱动：下载 → 解包 → 过滤 → 刷入

```
下载 zip（或本地选 zip）
      │
      ▼  extractDriverArchive()  递归解包，带 zip-slip 防护
filesDir/drivers/<id>/           并写入 .source 标记（来源 URL / local:<zip名>）
      │
      ▼  detectKernelInfo() 读 os.version（不用 root），解析成 "6.6"
      │  filterByKernel()   只看主次版本匹配的文件，可切“显示全部”
      ▼
flashDriverScript()              su 执行：
                                 mkdir -p /data/han/drivers
                                 cp 文件 → chmod 777 → 执行（ELF 直接跑 / 脚本用 sh）
                                 date > <文件名>.installed   ← 刷入标记
```

- 每个驱动包对应一个 `DriverPackage`，每个包一份缓存目录，**只下载/解包一次**；
- `.source` 里存了来源链接，**换链接（换域名或换 id）会自动作废旧缓存**，不会一直显示上一版内容；
- 刷入时间靠 `/data/han/drivers/<name>.installed` 记录，不重复执行脚本来检测；
- 卡片上的「查看完整内容」用 `readDriverContent()` 逐行处理：去掉 ANSI 转义，把二进制行整行替换成占位说明，所以「脚本 + 内嵌二进制」的驱动包也能看到可读部分（上限 400KB）。

### 3. 终端与自动输入

- `PtyTerminal`：原生 PTY（`forkpty`）优先，失败退化管道 shell；接口一致（`write` / `resize` / `close` / `isRunning`）；
- `KernelAutoInput`：滑动窗口 4000 字符，识别「行首编号菜单」或「请选择 / 请输入 / Select」类提示，按用户配置的步骤依次写入；
- 步骤内容由用户自定义（`KernelInputStep(value, maxOption)`），`maxOption` 只是检测提示范围；**第一个内容为空的步骤及其之后都不会自动输入**；
- 回答完会 `window.clear()`，避免上一条提示把下一步一起触发。

### 4. 隐私处理

终端和驱动卡片都是「客户可见面」，如果下载源是自己的私有域名，界面里不该出现它：

- `hidePrivateLinks()` 会把包含 `PrivateDownloadHosts` 的整条链接、以及不带 `http://` 的裸域名统一替换成 `（下载地址已隐藏）`；
- 只抹自家域名，`https://example.com/a?b=1` 这类其他站点保持原样；
- 所有终端输出的入口 `appendLine()` 都过这一层，驱动卡片报错也用同一函数处理。

---

## 四、换下载链接 / 换服务器（最常改的地方）

> 还没有自己的下载源？跳到 **[第五节](#五下载源怎么来自建网盘推荐-r2-cloud-drive)**，用 R2-Cloud-Drive 自建一个，拿到的直链格式和本项目完全一致。

### 1. 内核链接

文件：`app/src/main/java/com/example/myapplication/MainActivity.kt`（文件顶部，约第 110 行）

```kotlin
// 内核获取链接
private const val KernelUrl =
    "https://域名/api/share-download?id=7c71a3d0cde94a10ab57c14834babff9"
```

改成任何**能直接返回文件内容**的地址即可，例如：

```kotlin
private const val KernelUrl = "https://example.com/kernel/cnhk.sh"
// 或者你自己的分享接口
private const val KernelUrl = "https://example.com/api/share-download?id=xxxx"
```

要求与注意：

- 返回 200 且是文件本体（脚本 / ELF / 无后缀 zip 都支持，按文件头识别）；
- 走 HTTPS 最省事；如果只能用 **明文 HTTP**，必须在 `app/src/main/res/xml/network_security_config.xml` 里加白名单，否则会被 Android 拦截：

  ```xml
  <domain-config cleartextTrafficPermitted="true">
      <domain includeSubdomains="false">你的域名或IP</domain>
  </domain-config>
  ```

- 分享接口边生成边下发时首字节较慢，`downloadKernelScript()` 的 `readTimeout` 是 **60 秒**，别随手调小；
- 如果服务端返回的是 zip，`resolveKernelScript()` 会自动解包并优先挑 ELF。

### 2. 驱动包列表

文件：`app/src/main/java/com/example/myapplication/Driver.kt`（约第 44~65 行）

```kotlin
private const val DriverShareHost = "https://域名/api/share-download?id="

val DriverPackages = listOf(
    DriverPackage(
        id = "RTDEV",
        name = "RTDEV",
        caption = "RT 驱动 · 16 个内核脚本",
        url = DriverShareHost + "a1ee4f083d6d4166b7f0ceaa3f2dae30"
    ),
    // ...
)
```

- `id` 同时是**缓存目录名**（`filesDir/drivers/<id>`），不同包必须不同；改 `id` 等于换缓存；
- `url` 可以是任意 zip 直链，不必用分享接口；
- 包内结构随便：有子目录、平铺在根目录都能解；只收 `.sh` / `.txt` / ELF；
- 想加一个包就在 `listOf(...)` 里追加一项，界面会自动多一张驱动卡片；
- 换链接后旧缓存会自动作废（`.source` 标记比对 url），无需手动清数据。

### 3. 域名隐藏列表（换域名后必改）

如果新链接还是私有域名，且不希望它出现在终端/界面上：

文件：`MainActivity.kt`，`hidePrivateLinks()` 上方

```kotlin
private val PrivateDownloadHosts = listOf("915161.xyz")
```

改成你的域名（支持多条），例如：

```kotlin
private val PrivateDownloadHosts = listOf("example.com", "cdn.example.com")
```

或者如果你打算完全公开下载源，也可以直接删除这一层（把 `hidePrivateLinks` 改成 `text` 原样返回），界面上就不会再有「（下载地址已隐藏）」。

### 4. 应用名、包名、图标

| 想改什么 | 位置 |
| --- | --- |
| 桌面显示的名字 | `app/src/main/res/values/strings.xml` → `app_name`（当前 `xtby`） |
| 安装包标识 | `app/build.gradle.kts` → `applicationId`（当前 `com.tencent.android.qqdownloader`） |
| Kotlin 包名 / namespace | `app/build.gradle.kts` → `namespace`，以及源码目录与 `package` 声明 |
| 版本号 | `app/build.gradle.kts` → `versionCode` / `versionName` |
| 图标 | `res/mipmap-*`、`res/drawable/ic_launcher_*` |
| 根工程名 | `settings.gradle.kts` → `rootProject.name` |

> 开源自建版强烈建议把 `applicationId` 和 `namespace` 换成你自己的（例如 `com.yourname.xtby`），否则会和别人的包冲突，也无法上架/签名分发。

### 5. 联系方式 / 关于页

`MainActivity.kt` 的 `AboutScreen()`（约第 2127 行）：TG 群 `https://t.me/BYYXnb`、QQ 群 `427975011` 都是硬编码字符串，开源前记得替换或删掉。

### 6. 运行路径常量（一般不用改）

`MainActivity.kt` 顶部：

```kotlin
private const val KernelDirectory        = "/data/han"
private const val KernelFileName         = "cnhk.sh"
private const val DriverRootDirectory    = "/data/han/drivers"
private const val KernelPayloadFileName  = "cnhk.sh.payload"
```

如果改成别的目录，`kernelPickFragment()` / `kernelRunFragment()` / `flashDriverScript()` 都读这些常量，不用逐处改。

---

## 五、下载源怎么来：自建网盘（推荐 R2-Cloud-Drive）

本应用对「下载源」的要求极低：**一个 GET 请求就能返回文件本体的 HTTPS 地址**。所以对象存储、静态托管、自建网盘都行，不需要任何私有 API。

这里推荐一个可以自己部署的免费方案：

> **[HandsomeMJZ/R2-Cloud-Drive](https://github.com/HandsomeMJZ/R2-Cloud-Drive)** —— 基于 Cloudflare Workers + R2 的云端硬盘，Material Design 风格，单文件 `worker.js`，不用构建。

选它的理由：它的**分享下载直链格式，正好就是本项目现在用的那种**：

```
https://你的域名/api/share-download?id=分享后缀
```

也就是说，你把它部署起来之后，只要把 `KernelUrl` 和 `DriverPackages[].url` 换成自己域名下的直链，**其余代码一行都不用改**。

### 1. 它自带的能力（对本项目有用的部分）

- 文件上传 / 下载 / 重命名 / 新建文件夹，网格 / 列表视图、夜间模式；
- **自定义分享链接**：`/s/自定义后缀`，可设密码、有效期、访问次数；
- **免密码单文件分享可「复制下载直链」**：形如 `/api/share-download?id=my-file`，无需先打开分享页 → 这就是 `HttpURLConnection` 直接拉文件要的地址；
- `/shared` 目录：公开只读，适合放「大家都能下」的公共包；对应直链是 `/api/download?path=shared%2F文件名`（`path` 需 URL 编码）；
- 大文件分片上传 / Range 分段下载，支持多 Cloudflare 账号的多个 R2 节点合并容量；
- 可选 `ACCESS_PASSWORD` 保护后台，`SITE_TITLE`、`CLOUD_ICON_URL`、`LOGIN_BACKGROUND_URL` 可换皮肤。

### 2. 部署步骤（官方文档的精简版）

1. Cloudflare 建 **R2 存储桶**（注意：Cloudflare 账号需绑定 PayPal 才能创建 R2）；
2. **Workers & Pages → D1** 建一个数据库；
3. 新建 **Worker**，把仓库里的 `worker.js` 全文粘进在线编辑器，Deploy（无需构建，升级也是替换全文再 Deploy）；
4. **Settings → Bindings** 加两个绑定，名字必须完全一致：
   - `R2_BUCKET` → 你的 R2 桶
   - `DB` → 你的 D1 库（表结构由 Worker 首次访问时自动创建）
5. （可选）Variables 里设 `ACCESS_PASSWORD` 等；
6. 得到 `https://项目名.账号.workers.dev`，或 **Triggers → Custom Domains** 绑自己的域名。

详细步骤、多节点、API 说明请看仓库里的 `README.MD`、`API说明文档.MD`、`TECHNICAL_DOC.MD`。

### 3. 拿到直链并填进本项目

1. 上传内核文件（脚本 / ELF / 无后缀 zip 都行）和驱动包 zip；
2. 选中文件 → **分享设置** → 填后缀（1~64 位字母、数字、`_`、`-`，字母或数字开头，会转小写）；
3. **不要设密码**（应用侧不做登录/密码流程），保存后点 **复制下载直链**：

   ```
   https://你的域名/api/share-download?id=my-kernel
   ```

4. 填进代码：

   ```kotlin
   // MainActivity.kt 顶部
   private const val KernelUrl =
       "https://你的域名/api/share-download?id=my-kernel"
   ```

   ```kotlin
   // Driver.kt
   private const val DriverShareHost = "https://你的域名/api/share-download?id="

   val DriverPackages = listOf(
       DriverPackage(
           id = "MYDRIVER",                 // 只用于本地缓存目录名
           name = "我的驱动",
           caption = "自定义驱动包",
           url = DriverShareHost + "my-driver"
       )
   )
   ```

### 4. 几个容易踩的坑

- **后缀字符集**：只能用英文字母、数字、`_`、`-`。别用中文或空格，否则 URL 编码后不好维护；
- **改后缀 / 刷新链接会让旧地址失效**。换完必须同步改 App 里的常量并发版，否则老版本客户端直接拿到 404（本应用会显示「下载失败：HTTP 404」）。建议在 README 的版本说明里记一下「本次链接变更」；
- **驱动包建议「一个包一个后缀」**，不要用目录分享：目录分享的 `id` 指向目录，客户端下下来不是 zip；
- **有效期 / 访问次数**：直链同样受分享的有效期和访问次数限制。给 App 用的链接建议**不设次数上限**，否则用户量一大就集体下不动；
- **`ACCESS_PASSWORD` 只保护网页后台**，不影响直链，但仍受上面那两条限制；
- **域名隐藏**：换域名后记得同步 `PrivateDownloadHosts`（见上一节 4.3），否则终端里会打出你的域名；
- **HTTPS 够用**：`workers.dev` 和自定义域名都是 HTTPS，无需改 `network_security_config.xml`；只有用明文 HTTP 自建站时才需要加白名单；
- **合规提醒**：该仓库目前在 GitHub 上**未标注开源许可证**（license 显示为空）。自建部署（把 worker.js 跑在自己的 Cloudflare 账号里）通常没问题，但**二次分发或商用前请先确认作者授权**，或到仓库提 issue / 通过作者的 TG 联系。相比之下本项目你自己选的许可证（建议 Apache-2.0 / MIT）要写在 `LICENSE` 里。

### 5. 其它可选方案

| 方案 | 适合场景 | 备注 |
| --- | --- | --- |
| GitHub Releases 直链 | 几乎零成本、包不大 | `releases/download/<tag>/<file>` 可直接 GET；注意大陆网络可达性 |
| R2 公共桶 + 自定义域名 | 想完全掌控、用 Cloudflare | 桶设为公开，对象直链即文件地址；无后台 UI |
| 任意对象存储（S3 兼容 / 七牛 / 又拍等） | 已有存量 | 生成公共读 URL 即可 |
| 自建 Nginx / Caddy 静态目录 | 已有 VPS | 必须配 HTTPS，否则要加明文白名单；只放「可公开」的文件 |

> 无论用哪种，都建议**内核和驱动分开托管目录**：内核换得勤，驱动包体积大，混在一起不好做缓存和失效。

---

## 六、权限与前置条件

- `AndroidManifest.xml` 只申请了 `INTERNET`；
- 所有落盘到 `/data/han`、刷驱动的操作都通过 **`su -c`** 执行，即**设备必须已 root 并授予应用 root 权限**；
- 应用自身不请求存储权限：内核/驱动文件下载到 `cacheDir`，解包到 `filesDir`，本地导入走 SAF（`OpenDocument`，MIME `*/*`）；
- 解析设备内核版本只用 `System.getProperty("os.version")`（不需要 root），拿不到时回退读 `/proc/version`。

---

## 七、常见问题

**Q：终端说「PTY 原生库不可用，已改用管道 shell」**
A：APK 里没有 `libptyterminal.so`（构建时没跑 native/NDK，或设备 ABI 不是 arm64-v8a）。功能可用，只是没有 tty，交互式菜单可能不响应。要修就确保构建时执行了 CMake 任务，并在 `externalNativeBuild` 保留 `abiFilters`。

**Q：下载成功但内核「怎么都跑不起来」、终端没有任何输出**
A：多半是内核是自解压脚本，而设备缺 `gzip`/`mktemp` 或 `TMPDIR` 不可写。本工程会自动解出 `.payload` 并优先执行它；如果 `prepareKernelScript()` 的日志里没有「自解压负载已解开」，说明文件不是标准的 `#!` + gzip 结构，需要看服务端给的到底是什么。

**Q：换了自己的链接，界面还显示旧驱动**
A：`filesDir/drivers/<id>` 已有缓存时不会自动重下。换 `url` 会自动作废；只换服务器但 url 不变的话，点卡片上的重新下载（或在系统里清除应用数据）。

**Q：明文 HTTP 链接下不下来**
A：targetSdk 28+ 默认禁止明文流量，需要改 `network_security_config.xml`。注意 `<application android:networkSecurityConfig>` 已指向该文件。

**Q：`downloadKernelScript` 里 `use` / `responseCode` 报 Unresolved reference**
A：那是文件末尾自定义的 `HttpURLConnection.use` 扩展，部分分析器会误报，编译是正常的。

---

## 八、开源前检查清单

- [ ] 替换 `KernelUrl`、`DriverPackages[].url`（或删掉你不拥有的驱动包）
- [ ] 确认下载源已部署好（见第五节，自建网盘 / 对象存储 / Releases），并在 README 里写清「链接怎么换」
- [ ] 给 App 用的分享直链**不设密码、不设访问次数上限**，否则用户会集体下载失败
- [ ] 更新 `PrivateDownloadHosts`，或移除隐藏逻辑
- [ ] 替换 `applicationId` / `namespace` / `app_name` / 图标
- [ ] 清理 `AboutScreen` 里的私人联系方式
- [ ] 确认 `network_security_config.xml` 不残留旧 IP/域名
- [ ] 加上 `LICENSE`（如 Apache-2.0 / MIT）与 `signingConfig`
- [ ] 如果你要分发内核/驱动文件本身，确认你有相应授权

---

## 九、免责声明

本项目只是一个执行壳：不内置、不修改、不担保任何内核或驱动。刷写内核/驱动、获取 root、执行第三方脚本均有**变砖、数据丢失、失去保修**的风险，请自行评估并承担后果。请遵守你所在地区的法律法规以及相关软件/服务的许可协议。
