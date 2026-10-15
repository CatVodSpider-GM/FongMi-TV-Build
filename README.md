# FongMi-TV-Build

FongMi TV（Android 客户端）的打包仓库。**这里不放源码**——workflow 运行时会去
[FongMi/TV](https://github.com/FongMi/TV) 拉取指定 ref 再编译，产物直接发到本仓库的 Release。

## 用法

Actions → **Build FongMi TV APK** → Run workflow。两个可选参数：

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `ref` | `fongmi` | 要编译的分支 / tag / commit SHA |
| `tag` | 空 | Release tag，留空则用 `v<当天日期>`，例如 `v2026.10.15` |

同名 tag 的 release 会被更新、同名 asset 会被覆盖，所以同一天重复跑是安全的（后一次覆盖前一次）。

产物（`leanback` / `mobile` × `arm64-v8a` / `armeabi-v7a`）：

```
leanback-arm64_v8a.apk      电视盒子 64 位
leanback-armeabi_v7a.apk    电视盒子 32 位
mobile-arm64_v8a.apk        手机 / 平板 64 位
mobile-armeabi_v7a.apk      手机 / 平板 32 位
```

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
