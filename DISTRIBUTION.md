# 分发说明（dist 分支）

本文件说明本分支与上游的差异、分发时的合规义务和已知风险。**对外发布前请完整读一遍。**

---

## 1. 这是什么

一个**第三方、非官方**的短剧 Android 客户端，基于上游 `OOIUa/hongguo-android` 的 GPL-3.0-only 源码，额外合入了弹幕功能，并做了去商标化处理以便独立分发。

来源链（每一层都是公开的 GPL 项目）：

| 层 | 项目 | 内容 |
| --- | --- | --- |
| 上游客户端 | `OOIUa/hongguo-android` | 播放器、接口协议、Compose UI |
| 弹幕分支 | `heyvye/hongguo-android` 分支 `feature/danmaku`（对应上游 PR #10） | 弹幕层、设置面板、播放页开关 |
| 弹幕原始实现 | `jycao2/hongguoTV`（commit `09019a3`） | `danmaku/` 五个 Kotlin 文件的移植来源 |

本分支在 `feature/danmaku` 之上只做了分发相关改动，**没有改动弹幕算法本身**。逐项差异见下方第 2 节，每一处都在提交说明里写清了目的。

## 2. 相对上游的改动

| 项 | 上游 | 本分支 | 原因 |
| --- | --- | --- | --- |
| `applicationId` | `com.hongguo.android` | `com.hongguo.android.dm` | 与上游版并存、互不覆盖；避免顶着上游应用身份分发 |
| 应用名（桌面/我的） | 红果独立版 / 红果剧场 | 弹幕剧场 | 不使用他人商标作为应用标识 |
| 启动图标与页头标记 | 上游品牌素材 | 自绘「对话气泡 + 播放键」矢量 | 同上；未复制任何上游美术素材 |
| 在线更新源 | 固定指向上游仓库 | 由 `-PupdateRepository=<owner>/<repo>` 指定，默认不绑定 | 否则会提示一个签名不符、装不上的「新版」 |
| 弹幕 | 自 0.7.32 起内置官方弹幕 | 官方弹幕 + **字体切换**（B站 = 内置思源黑体，默认；鸿蒙 = 内置 HarmonyOS Sans；系统 / 宋体 / 等宽） | 本分支的存在理由 |
| 内置字体 | 无 | 思源黑体（OFL 1.1）+ HarmonyOS Sans（华为字体许可），均未修改字形，许可全文在 `assets/licenses/` | 字体许可要求显著声明，见 `THIRD_PARTY_NOTICES.md` |

`namespace`、包内所有 `com.hongguo.android.*` 的类名和接口协议**都没有改**，只有 `applicationId` 变了。这是刻意的：改动面越小，越容易让别人复核这份代码。

账号相关文案里保留的「红果账号 / 红果风控拒绝」等字样，是在**描述所对接的平台**，属于指称性使用，不是把该名称当作本应用标识。这类文案不要改成别的名字，否则会误导用户以为是另一个平台的账号。

## 3. 安装与升级

- 与上游版**可以并存**（包名不同），但**不能互相覆盖安装**：两者签名证书不同。
- 本分支内部升级：只要签名证书不变，可以正常覆盖安装，保留本机收藏、历史与续播位置。
- **不能覆盖安装**时的表现是 `INSTALL_FAILED_UPDATE_INCOMPATIBLE`。此时只能卸载重装，卸载会删除本机数据（设备身份、收藏、历史、续播位置、私有缓存）。这是有损操作，不能对外描述为「无损升级」。

## 4. 签名

本分支当前使用 **Android debug 签名**。这一点必须让使用者知道：

- 证书：`CN=Android Debug`，SHA-256 `a230559be2d904fcc0e3ccefe68730156e84a45a2f7f9603999efe3283a56f55`（构建机：`~/.android/debug.keystore`）。
- 与上游发布签名（`release-signing.sha256` 记录的 `d06122bd…`）**不是同一把密钥**。该文件属于上游，本分支不要改写它来「对齐」——那会掩盖签名变化。
- **debug keystore 必须备份**。换机器、重装 SDK、删除 `~/.android` 都会生成新 key，届时老用户无法覆盖升级，只能卸载。若要长期发布，建议改用一把自己生成并离线保管的固定 keystore（把 `.jks` 放在仓库外，用 `-P` 或环境变量传入），而不是依赖 debug keystore。
- 每次发布前用 `apksigner verify --print-certs <APK>` 核对证书指纹，指纹变了就停下来找回密钥。

## 5. GPL-3.0-only 义务

本项目整体是 GPL-3.0-only。**分发 APK 就等于分发目标代码，必须同时提供对应源码。**

- 源码就是本仓库。发布时给每个版本打 tag（如 `v0.7.33-dm24`），Release 说明里写明「对应源码：tag `v0.7.33-dm24`」，或直接把 `source.zip` 传成附件。
- 保留 `LICENSE`、`THIRD_PARTY_NOTICES.md`、`app/src/main/assets/licenses/` 下的全部许可文本，不要删。
- 修改过的文件要在提交记录里说明改了哪里（见第 2 节表格），这是 GPL §5 对「标记修改」的要求。
- 不得对 GPL 代码附加额外限制（不能加「禁止商用」「禁止再分发」之类的条款）。
- 如果要分发二进制但不给源码，或者只在私有渠道发 APK 不给源码，就不符合 GPL，请不要这样发。

## 6. 发布步骤

```bash
# 1) 构建（JAVA_HOME 必须是 17+）
export JAVA_HOME="D:\\android-build\\jdk"
export ANDROID_HOME="D:\\android-build\\sdk"
printf 'sdk.dir=D:/android-build/sdk\n' > local.properties      # 不要提交这个文件
./gradlew :app:assembleDebug \
  -PappDanmakuBuild=23 \
  -PupdateRepository=<你的账号>/<仓库名>

# 2) 核对（四项版本信息 + 签名 + 弹幕类真的进了 dex）
apksigner verify --print-certs app/build/outputs/apk/debug/HGMini-v0.7.33-dm24-debug.apk
aapt2 dump badging app/build/outputs/apk/debug/HGMini-v0.7.33-dm24-debug.apk | head -3
apkanalyzer dex packages app/build/outputs/apk/debug/HGMini-v0.7.33-dm24-debug.apk | grep danmaku

# 3) 出哈希
sha256sum app/build/outputs/apk/debug/HGMini-v0.7.33-dm24-debug.apk
```

发布检查单：

- [ ] tag 与 `versionName` 一致（`v0.7.33-dm24` ↔ `0.7.33-dm24`），`versionCode` 严格递增（73324）
- [ ] Release 说明写明：对应源码 tag、APK 的 SHA-256、包名、minSdk、签名指纹、本次改动
- [ ] 若指定了 `updateRepository`，把 `update.json` 一起传到 Release（`latest/download/update.json` 路径要能取到），否则应用内更新会失败
- [ ] 若没指定 `updateRepository`，Release 说明里写「本包未绑定更新源，请手动下载新版」
- [ ] 真机至少验证：安装、播放、弹幕开启、暂停、切集、杀进程重启后参数保留
- [ ] `VALIDATION.md` 追加本次实测记录

## 7. 不要提交 / 不要上传

`local.properties`、`*.jks` / `*.keystore` / `*.p12`（私钥与口令）、`build/` 产物、构建日志、任何 GitHub 凭据、任何账号 cookie / token / 设备身份。`.gitignore` 已覆盖主要项，提交前仍请 `git status` 过一眼。

APK 只作为 Release 附件，不进 git 历史（26 MB 的二进制会把仓库撑坏）。

## 8. 风险与免责（必须写进 Release 说明）

- **非官方**：本项目与红果 / 抖音 / 番茄小说及其关联公司**没有任何关系**，未获授权，不代表其立场。
- **接口随时可能失效**：客户端靠逆向官方接口工作（含自签 `Signer`、匿名设备注册）。官方改版即可能静默失效，表现为播放失败或弹幕不出现，不会崩溃。
- **有风控**：弹幕与播放都走官方接口，可能被限流或拒绝。失败时是「退避重试后放弃」，不是 bug 修复能解决的。
- **不要开全局 VPN**：上游 FAQ 记载 VPN 会导致设备注册失败，播放和弹幕都取不到。
- **建议不要使用主账号登录**。
- **不含片源**：本仓库不分发任何视频内容；内容版权归原权利人。分发者不因此获得任何内容授权。
- **不要做规避**：不要为了「防封」「防检测」改动请求特征、伪造更多身份或加入对抗性逻辑。那会把一个可讨论的兼容性问题变成另一个性质的问题，也会让这份交付无法被复核。

## 9. 已知未验证项

模拟器（AVD `hg36`，Android 16，x86_64）已验证，**真机未验证**：横竖屏切换、画中画、手势层与弹幕层的叠放、悬浮播放、不同厂商 ROM 的后台限制、账号登录全流程。
