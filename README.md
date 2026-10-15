# FongMi-TV-Build

FongMi TV（Android 客户端）的打包仓库。**这里不放源码**——workflow 运行时会去
[FongMi/TV](https://github.com/FongMi/TV) 拉取指定 ref 再编译，产物直接发到本仓库的 Release。

## 用法

Actions → **Build FongMi TV APK** → Run workflow。四个可选参数：

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `ref` | `fongmi` | 要编译的分支 / tag / commit SHA |
| `media3_ref` | `3c2cbe8ac…` | `FongMi/media` 上编 20 个 `lib-*.aar` 用的 ref |
| `libass_ref` | `9dae39e968…` | `FongMi/media` 上含 `exoplayer_libass` 的 ref（与前者已 diverged） |
| `tag` | 空 | Release tag，留空则用 `v<当天日期>`，例如 `v2026.10.15` |

同名 tag 的 release 会被更新、同名 asset 会被覆盖，所以同一天重复跑是安全的（后一次覆盖前一次）。

产物（`leanback` / `mobile` × `arm64-v8a` / `armeabi-v7a`）：

```
leanback-arm64_v8a.apk      电视盒子 64 位
leanback-armeabi_v7a.apk    电视盒子 32 位
mobile-arm64_v8a.apk        手机 / 平板 64 位
mobile-armeabi_v7a.apk      手机 / 平板 32 位
```

## 为什么要拉三个仓库（`lib-*.aar`）

`FongMi/TV` 的源码里有 291 处 `androidx.media3.*` import，但**任何 gradle 文件都没声明 media3**——
唯一入口是 `app/build.gradle` 的 `implementation fileTree(dir: "libs", include: ["*.aar"])`，
补上那些类的是 21 个 `lib-*-release.aar`。而根 `.gitignore` 和 `app/libs/.gitignore` 都写着 `lib-*.aar`，
上游 README 也明说「配套 AAR 放入 `app/libs/`，`lib-*.aar` 未納入 Git，單純 clone 不包含完整播放器依賴」。

**所以裸 clone 一定编不过**，报的就是 `package androidx.media3.* does not exist`。
那 21 个 AAR 是 FongMi 从自己的 media3 fork 编出来的（他本机靠 `FongMi/media/move.bat` 从
`D:\Project\media\libraries` 拷进 `D:\Project\TV\app\libs`，`move.txt` 里列了 20 个文件名）。
workflow 就是把这条链路搬到 CI 上：

1. `FongMi/media` @ `media3_ref` → 按 `media3/move.txt` 编 **20 个** `lib-*-release.aar`
   （ffmpeg / mpv / mpegh 的预编译 `.so` 都在仓库里，不用自己编 native）；
2. `FongMi/media` @ `libass_ref` → `prepare_libass.sh` 现编 native，产出 `lib-exoplayer-libass-release.aar`
   （这个模块不在 `media3_ref` 那条分支上，两者已 diverged，所以得 checkout 两份）；
3. 把 21 个 AAR 塞进 `app/libs/`，再编 TV。

如果哪天 `move.txt` 的条数变了、或 `lib-*.aar` 没编出来，构建会以
`缺少 media3 AAR: …` / `只拷了 N 个 AAR，应该是 M 个` 直接失败——那是 workflow 的断言，不是 media3 的问题。

## 签名

`app/build.gradle` 的 release 构建走 `signingConfigs.release`，签名信息从 **`local.properties`**
读（`storeFile` / `keyAlias` / `storePassword`，注意 key 密码也取 `storePassword` 的值）。
workflow 每次都会现场生成这个文件。

想用固定签名（这样新包能覆盖安装旧包）就配下面三个 secret：

| Secret | 说明 |
| --- | --- |
| `KEYSTORE_BASE64` | keystore 文件的 base64：`base64 -w0 release.jks` |
| `KEYSTORE_PASSWORD` | store 密码，**同时也是 key 密码** |
| `KEY_ALIAS` | key alias |

配好 keystore 时记得是 `keyPassword == storePassword`，否则签名会失败。

> 一个 `local.properties` 的坑：`Properties.load()` 会把 `\` 当转义符，密码里别用反斜杠。

**不配 secret 也能跑**，此时会临时生成一个随机 keystore 并打 `::warning::`。
代价是每次构建签名都不同，装过旧包必须先卸载才能装新包，不适合日常自用。

## 构建环境

| 项 | 值 |
| --- | --- |
| Gradle | 9.7.1（wrapper） |
| AGP | 9.3.1 |
| JDK | 21 (temurin) |
| compileSdk / targetSdk | 37 / 37，minSdk 24 |
| Python | 3.10（chaquopy，`chaquo/requirements.txt` 里的包会现装） |
| NDK / CMake | 29.0.14206865 / 3.22.1（给 libass 编 native 用） |
| 原生工具链 | autoconf / automake / libtool / ninja / pkg-config（`prepare_libass.sh` 会对每个依赖跑 autogen） |

Chaquopy 有两套 Python：**target** 是打进 APK 的那份（`chaquo/build.gradle` 的 `python { version = "3.10" }`），
**buildPython** 是构建机上跑 pip 的那份，主次版本必须和 target 一致。runner 自带的是 3.12+，
所以 workflow 里专门用 `actions/setup-python` 装了 3.10，否则 `installReleasePythonRequirements`
会以 `Couldn't find Python 3.10` 失败。

另外上游 `FongMi/TV` 把 `gradlew` 提交成了 `100644`（没有可执行位），Linux 上直接 `./gradlew`
会 `exit 126`；workflow 里先 `chmod +x`。注意在 Windows 上 `ls -l gradlew` 会假显示 `-rwxr-xr-x`，
要看真实 mode 得用 `git ls-tree`。

`media3` / `media3-ass` 两份检出用的是**它们自己的** wrapper（media3 是 Gradle 9.1.0），
所以那两步是 `./media3/gradlew -p media3 …`，跟 TV 的 `./gradlew` 互不干扰。

`compileSdk 37` 不一定在 runner 预装的 SDK 里，workflow 用 `android-actions/setup-android`
装好 cmdline-tools 并接受 license，剩下的 platform / build-tools 交给 AGP 自动下载。

## 加自动构建

默认只支持手动触发。想让上游更新后自动出包，把 workflow 里那段注释掉的 `schedule:` 打开即可。

## 关于 Chaquopy 的 ABI 拆分

`buildSrc/.../AbiApkPackaging.groovy` + `FinalizeApksTask.groovy` 做了两件非常规的事：

1. 打包后按 ABI 拆出各自的 APK，并**删掉不属于该 ABI 的 chaquopy 资源**
   （`assets/chaquopy/{requirements,stdlib}-<abi>.imy`、`bootstrap-native/<abi>/`）；
2. 删完重跑 `zipalign` 并用 `apksigner` 重新签名（只开 v2）。

所以如果哪天 chaquopy 改了资源命名，构建会以
`Incomplete Chaquopy ABI assets for <abi>` 直接失败——那是这套脚本的断言，不是 chaquopy 本身的问题。
