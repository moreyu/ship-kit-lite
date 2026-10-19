---
name: app-store-review-precheck
description: 提审前逐条自检 App Store 审核风险的技能，适用于发布 iOS/macOS App 的 AI 编程场景。触发场景：准备向 App Store Connect 提交构建、填写版本元数据、创建内购或订阅、配置测试账号与审核备注，或已收到拒审邮件需要逐条定位（Guideline 2.1 App 完整性、2.3 元数据准确、3.1.1 数字内容须用内购、3.1.2 订阅信息披露、4.2 最低功能与纯 WebView 套壳、4.3 重复 App、5.1.1 隐私政策链接与 App 内账号删除、PrivacyInfo.xcprivacy 与 Required Reason API、第三方 SDK 隐私清单）时使用。关键词：App Store 审核、过审、拒审、被拒、Guideline、提审前检查、Info.plist 权限用途字符串 NS*UsageDescription、UserDefaults、systemUptime、文件时间戳、恢复购买、外链支付、账号注销、隐私政策 URL、审核备注与 demo 账号、中国区备案号。Agent 用 rg 扫描 Xcode 工程并按条款编号输出可执行的修复清单。
---

# App Store 提审前自检（app-store-review-precheck）

## 1. 目的

把「用 AI 写出来的 SwiftUI 原型」变成「敢点提交的构建」。本技能只做三件事：

1. **用证据说话**：用 `rg` 在本地仓库扫描真实代码与配置文件，每条结论都要落到「文件路径:行号」或「App Store Connect 具体页面」，不靠印象判断。
2. **映射条款编号**：把证据对上 Guideline 编号（2.1 / 2.3 / 3.1.1 / 3.1.2 / 4.2 / 4.3 / 5.1.1）以及 privacy manifest（`PrivacyInfo.xcprivacy`）与 Required Reason API 的检查面。
3. **产出可执行报告**：输出 Markdown 报告，分「阻塞项（不改大概率被拒）」「风险项」「必须由用户在 ASC 人工确认的项」，每项给出具体改法。

**非目标（不要越界）**：不承诺过审结果；不逐字引用 Apple 准则原文（只写编号 + 确定的要点）；不在用户没要求时改业务代码；不替代中国区 APP 备案流程（完整版的 `china-app-filing` 技能负责；Lite 版只检查备案号是否已填、是否一致）；不替用户决定定价与商业模式。

**执行前必读**：`references/guidelines-cheatsheet.md`（常见拒审点速查表：症状 / 如何用 rg 查 / 修复方法 / 逐条 rg 命令）。它是本技能的执行细则，本文件是执行流程。

## 2. 输入

### 2.1 先自己读，不要问用户你自己能查到的东西

| 需要的东西 | 去哪读 |
|---|---|
| 工程根目录与目标名 | `*.xcodeproj/project.pbxproj`、`*.xcworkspace`、`Package.swift` |
| 是否用自动生成的 Info.plist | `pbxproj` 里的 `GENERATE_INFOPLIST_FILE`、`INFOPLIST_KEY_*`（Xcode 13+ 常见） |
| 权限用途字符串 | `**/Info.plist` 的 `NS*UsageDescription` 键 |
| 隐私清单 | 工程内是否有 `PrivacyInfo.xcprivacy`，且是否在 target 的 Copy Bundle Resources 里 |
| 内购/订阅 | 代码里的 StoreKit 调用、`*.storekit` 配置文件、`Package.resolved`/`Podfile` 中的 RevenueCat 等 |
| 第三方 SDK | `Package.resolved`、`Podfile.lock`、`Cartfile.resolved` |
| 账号与删除 | 登录实现（Sign in with Apple / Firebase Auth / Supabase / 自建）+ 是否有删号接口与 UI 入口 |
| 网络/后端 | 是否有真实后端域名、是否离线可用、是否有本地 mock 兜底 |

只有当范围内确实没有文件（例如用户只给了一段描述，或项目不在当前工作目录）时，才向用户索取路径。

### 2.2 必须向用户确认的输入（本地代码查不到的部分）

一次问清，别挤牙膏：

1. App 名称、Bundle ID、目标上架的地区（是否包含中国大陆）。
2. 是否包含**账号体系**；若有，苹果审核要用哪个 demo 账号、密码是否会过期、后端在审核期间是否 7×24 可用。
3. 是否收费、用哪种方式：IAP 买断 / 自动续订订阅 / 完全免费（决定 3.1.1、3.1.2 是否适用）。
4. 隐私政策 URL、支持 URL（必须是公开可访问的 https 页面，且 App 内能找到入口）。
5. App Store Connect 里「App 隐私」问卷是否已填、填了哪些数据类型（决定第三方 SDK 与 manifest 的一致性）。
6. 中国区是否需要备案号，以及是否已拿到（据第三方实践文章，App 备案号形如 `-1A` 结尾）。
7. 是否有特殊要求要在审核备注里说明：硬件依赖、地区限制、需要登录才能看到的核心功能、订阅测试方式。

如果用户无法提供 ASC 访问权限，就让他按「ASC 页面 → 字段名」截图或抄字段值回来，把这部分标成「待人工确认」。

## 3. 分步执行指令

按顺序执行，每步记录证据（路径:行号 或 ASC 页面/字段）。任何一步发现阻塞项，仍然走完整流程再汇总——不要提前中断。

### 步骤 0：确定扫描范围

- 确认工作目录就是 Xcode 工程根目录；如果不是，先 `cd`/指定 `--path`。
- 列出全部待扫文件：`rg --files | head -200`（先确认规模，避免扫进 `Pods/`、`build/` 等生成目录；扫描时统一加 `-g '!Pods/**' -g '!build/**' -g '!DerivedData/**'`）。
- 记录 Xcode/SDK 版本与目标的 Deployment Target（`pbxproj` 里的 `IPHONEOS_DEPLOYMENT_TARGET`），因为 SDK 与最低系统版本要求会随时间变化。

### 步骤 1：2.1 App 完整性——「能不能跑、够不够完整」

1. 真机/模拟器跑一次 Release 构建，走一遍核心流程（登录 → 主功能 → 付费墙 → 退出）。
2. 扫未完成的痕迹：`rg -ni 'TODO|FIXME|Lorem ipsum|敬请期待|coming soon|placeholder|测试版|beta' -g '!Pods/**'`。
3. 扫崩溃高风险写法：`rg -n '\btry!|\bas!\b|fatalError|preconditionFailure' -g '*.swift'`，以及强制解包 `rg -n '\.(first|last|max|min)!' -g '*.swift'`。
4. 检查异常路径：断网、空数据、权限被拒、后端 500 时是否有可用状态（而不是白屏或闪退）。
5. 检查审核备注：ASC →「我的 App」→ 版本页 →「App 审核信息」→「备注」/「登录信息」，必须写清 demo 账号、订阅测试路径、特殊硬件或地区限制。**留空 + 需要登录 = 2.1 高风险**。

### 步骤 2：权限用途字符串（Info.plist）与 5.1.1 权限用途

1. 列出 app 声明的全部权限键：
   `rg -o 'NS[A-Za-z]+UsageDescription' -g '**/*.plist' -g '*.pbxproj' | sort | uniq -c`
2. 逐个核对：`NSCameraUsageDescription`、`NSPhotoLibraryUsageDescription`、`NSPhotoLibraryAddUsageDescription`、`NSMicrophoneUsageDescription`、`NSLocationWhenInUseUsageDescription`、`NSLocationAlwaysAndWhenInUseUsageDescription`、`NSContactsUsageDescription`、`NSMotionUsageDescription`、`NSSpeechRecognitionUsageDescription`、`NSBluetoothAlwaysUsageDescription`、`NSCalendarsFullAccessUsageDescription`、`NSRemindersFullAccessUsageDescription`、`NSHealthShareUsageDescription`、`NSFaceIDUsageDescription`、`NSUserTrackingUsageDescription`、`NSLocalNetworkUsageDescription`。
3. 交叉验证：**用了 API 但没写 key → 运行时闪退（2.1）**；**写了 key 但代码里根本没用 → 会被问用途（5.1.1/2.3）**。查 API 是否真被调用：`rg -n 'AVCaptureDevice|PHPickerViewController|PHPhotoLibrary|CLLocationManager|CNContactStore|CMMotion|SFSpeechRecognizer|CBPeripheralManager|LAContext|ATTrackingManager|NSBonjourServices' -g '*.swift'`。
4. 若走的是自动生成 Info.plist，改的是 `pbxproj` 里的 `INFOPLIST_KEY_<Key>`，别只改一个不存在的 plist 文件。

### 步骤 3：PrivacyInfo.xcprivacy + Required Reason API

1. 存在性：`rg --files -g 'PrivacyInfo.xcprivacy'`。
2. 是否真进包：`rg -n 'PrivacyInfo' *.xcodeproj/project.pbxproj`——文件在磁盘上但没加进 target，等于没交。
3. 逐项扫描 Required Reason API 使用（对不上就补声明）：
   - UserDefaults：`rg -n 'UserDefaults|NSUserDefaults' -g '*.swift'`
   - 系统启动时间：`rg -n 'systemUptime|mach_absolute_time|clock_gettime' -g '*.swift'`
   - 文件时间戳：`rg -n 'creationDate|contentModificationDate|attributesOfItem|fileModificationDate' -g '*.swift'`
   - 磁盘空间：`rg -n 'volumeAvailableCapacity|NSFileSystemFreeSize|statfs|statvfs' -g '*.swift'`
   - 活跃键盘：`rg -n 'activeInputModes' -g '*.swift'`
4. 打开 `PrivacyInfo.xcprivacy`，核对四块：`NSPrivacyTracking`、`NSPrivacyTrackingDomains`、`NSPrivacyCollectedDataTypes`、`NSPrivacyAccessedAPITypes`（类别键形如 `NSPrivacyAccessedAPICategoryUserDefaults`）。类别覆盖了第 3 步命中的 API，且每个类别都有 Apple 定义的 reason 代码——**reason 代码必须查官方文档或 Xcode 生成的模板，不要凭记忆抄**（示例：UserDefaults 常填 `CA92.1`，需自行核对）。
5. 上传后被 Apple 邮件拦回来时，按邮件里的 ITMS 编号（例如隐私清单相关的 ITMS-91053 一类）逐条处理，以邮件原文为准。

### 步骤 4：3.1.1 数字内容必须走内购 + 恢复购买

1. 找到解锁逻辑：`rg -n 'StoreKit|Product\.products|\.purchase\(|Transaction\.currentEntitlements|AppStore\.sync' -g '*.swift'`。
2. 找外链支付/站外购买的痕迹（这是最常见的 3.1.1 死因）：
   `rg -ni 'alipay|wechatpay|weixin|stripe|paypal|checkout|扫码支付|请到网页购买|购买兑换码|激活码' -g '!Pods/**' -g '*.{swift,html,js,json,plist,strings}'`
   注意：App 内购买**数字内容/功能/去广告**必须走 IAP；实物商品、App 外服务才可以用第三方支付。少数地区/资格有 Apple 批准的 External Purchase Link 类计划，**不要当通用做法**。
3. 必须有恢复购买入口：`rg -ni 'restore' -g '*.swift'`，并确认按钮在付费墙/设置里真实可达（能找到函数不等于有入口）。
4. 产品 ID 与 ASC 一致性：ASC →「我的 App」→「订阅」/「App 内购买项目」页面，核对产品 ID、类型（消耗型/非消耗型/自动续订）、价格、本地化、审核截图都已填且状态不是「被拒绝」；本地 `*.storekit` 配置仅用于开发，不影响线上。

### 步骤 5：3.1.2 订阅信息披露

适用条件：App 含自动续订订阅（3.1.2(a)：订阅周期至少 7 天、在用户所有设备上可用；3.1.2(c)：订阅前说清付费得到什么，并满足开发者计划许可协议附表 2 的要求）。逐项在**付费墙代码**里核对，并对照 ASC 订阅页面：

1. 显示价格与周期（用 `product.displayPrice` 与周期文案，不写死「¥XX」）；
2. 说明自动续订（到期自动续订，除非提前取消）；
3. 说明如何取消（在系统「设置 → [你的名字] → 订阅」中管理）；
4. 有免费试用/优惠时，说明试用结束后的价格与扣费条件；
5. 付费条款（EULA）与隐私政策链接：`rg -ni 'terms|privacy|eula' -g '*.swift'`，链接必须可点开且不是 404；
6. 恢复购买入口（与 3.1.1 同一条）。

### 步骤 6：4.2 最低功能——套壳风险

1. 判断原生占比：`rg -n 'WKWebView|SFSafariViewController|loadRequest|loadHTMLString' -g '*.swift'`。如果这些命中是**主界面**，而原生功能只有登录和设置，就是 4.2 高危。
2. 找「比网页多出来的东西」：离线缓存、推送通知、相机/定位/健康、Widget、快捷指令、Apple Watch、iCloud 同步、本地数据模型——每项给出证据文件。
3. 检查首屏是不是只有欢迎页/占位页，核心内容全靠远端 HTML 下发；这种形态在 4.2 下非常脆弱。

### 步骤 7：4.3 重复 App（Spam）

1. 找模板残留：`rg -ni 'TEMPLATE|Copy of|MyApp|com\.example\.' -g '!Pods/**'`——Bundle ID、产品 ID、常量、字符串里残留 `com.example` 说明是套模板换皮，风险高。
2. 确认差异化：与开发者名下已有 App、以及 ASC 上同类 App 相比，功能/内容/交互是否有实质区别。若同一开发者账号下已有一堆仅换图标与文案的 App，4.3 是直接死因。
3. 检查名称是否蹭品牌词/其他 App 名（同时是 2.3 与商标风险）。

### 步骤 8：2.3 元数据准确

逐页核对 App Store Connect：

| ASC 页面 | 字段 | 检查点 |
|---|---|---|
| App 信息 | 名称 / 子标题 | ≤30 字符；中英文一致；不写「beta」「测试版」；不含价格类承诺 |
| App 信息 | 类别、内容版权 | 与真实内容一致 |
| 版本页 | 描述 | ≤4000 字符；不承诺未实现功能；不写其他平台（Android/网页版）的下载引导 |
| 版本页 | 关键词 | ≤100 **字节**（一个汉字 3 字节）；不含其他 App/公司名 |
| 版本页 | 支持 URL / 营销 URL | 可访问 |
| 版本页 | 截图 / App 预览 | 必须反映 App 真实界面；不能只有启动图或登录页；不能出现其他平台设备 |
| App 信息 | 隐私政策网址 | 与 App 内链接一致、可公开访问 |
| App 隐私 | 数据收集问卷 | 与实际采集、与 PrivacyInfo.xcprivacy、与第三方 SDK 一致 |
| App 信息 | 备案号（中国区） | 若含中国大陆：Apple 官方要求 ICP 编号与元数据和工信部备案一致（有简体中文以简体中文为准）；据第三方实践文章，备案号随下次提交生效、无备案号在中国区可能静默不展示；以官方与接入商最新要求为准 |

同时扫代码里的绝对化/禁止性文案：`rg -ni '第一|最好|#1|永久免费|破解|decrypt|无限下载' -g '!Pods/**'`。

### 步骤 9：第三方 SDK

1. 清单：`rg -n 'Firebase|Adjust|Amplitude|AppsFlyer|FBSDK|Sentry|OneSignal|RevenueCat|Umeng|Bugly' Package.resolved Podfile.lock Cartfile.resolved 2>/dev/null`。
2. 对 Apple「常用第三方 SDK」清单内的 SDK：必须使用**已包含隐私清单并已签名分发**的版本，版本过低可能直接上传失败或被拒。
3. 一致性：SDK 采集的数据类型必须同时出现在 `PrivacyInfo.xcprivacy` 与 ASC「App 隐私」问卷里，任何一处缺失都是 5.1.1/2.3 风险。

### 步骤 10：账号体系与 5.1.1 账号删除

1. 是否有账号体系：`rg -ni 'ASAuthorization|signInWithApple|FirebaseAuth|Supabase|Auth0|Cognito|LoginView|signIn\(' -g '*.swift'`。
2. 如有，必须有 **App 内**删除账号的入口与实现（只给「发邮件联系客服」不符合要求）：
   `rg -ni 'deleteAccount|delete_account|deleteUser|注销|删除账号|delete account' -g '!Pods/**'`。
3. 核对删除是否彻底：后端接口存在性、本地数据与 Keychain 清理、订阅引导（删号前提示需在系统订阅管理里取消）。
4. 隐私政策必须能从 App 内到达：`rg -ni 'privacy' -g '*.swift'`，且 URL 与 ASC 填写一致。

### 步骤 11：构建与上传前置

1. Release 归档构建通过（`xcodebuild -configuration Release` 或 Xcode Archive），无 error；warning 高危项记录在报告里。
2. 确认 SDK 与最低系统版本符合 Apple 当前上传要求（要求会变，见官方「Submitting apps to the App Store」页面，**提交前核对最新版**）。
3. ASC →「TestFlight」确认构建处理完成、无隐私清单/权限相关警告后，再进「版本页 → 构建版本」选择该构建。

## 4. 输出格式模板

报告写到工程根目录 `PRECHECK-REPORT.md`（或用户指定路径），严格用下面结构：

```markdown
# App Store 提审前自检报告

- 工程：<名称 / Bundle ID>
- 目标版本：<1.0 (1)>
- 扫描时间：<YYYY-MM-DD>
- 扫描范围：<目录，已排除 Pods/build/DerivedData>
- 结论：<可提交 / 修复后提交 / 暂不建议提交>

## 1. 证据汇总（按 Guideline）

| Guideline | 检查项 | 状态 | 证据（文件:行 / ASC 页面字段） | 修复动作 | 优先级 |
|---|---|---|---|---|---|
| 2.1 | 审核备注含 demo 账号 | 阻塞 | ASC 版本页 › App 审核信息 › 备注：空 | 补 demo 账号 + 后端可用性说明 | P0 |
| 3.1.1 | 站外支付入口 | 阻塞 | PaywallView.swift:88 `wechatpay` | 移除二维码，改走 StoreKit 产品 <ID> | P0 |
| 3.1.1 | 恢复购买入口 | 通过 | SettingsView.swift:42 | — | — |
| 5.1.1 | App 内删除账号 | 风险 | AccountView.swift 无删除入口 | 增加删号入口 + 调 DELETE /user | P1 |
| privacy | Required Reason API 声明 | 风险 | Store.swift:31 `UserDefaults` 未在 xcprivacy 声明 | 补 NSPrivacyAccessedAPICategoryUserDefaults + reason | P1 |
| 4.2 | WebView 套壳 | 通过 | 主界面为原生 SwiftUI，WebView 仅用于帮助页 | — | — |

状态取值只有三种：**通过 / 风险 / 阻塞**。

## 2. 阻塞项（P0，改完再提交）

1. <问题>｜证据：<路径:行号 / ASC 页面字段>｜改法：<具体一步到位>｜复核命令：`<rg 命令>`

## 3. 风险项（P1/P2）

1. <问题>｜证据｜改法｜复核命令

## 4. 必须由你在 ASC 人工确认的项

- [ ] <页面 → 字段>：<要确认什么>（本地代码无法判定）

## 5. 复检命令（改完自己跑一遍）

```bash
<逐条 rg 命令>
```

## 6. 免责声明

本报告基于本地代码与用户提供的 ASC 信息，不保证过审；条款编号对应的正式表述与最新要求请以 Apple 官方为准。
```

要求：每条结论都要有证据；查不到证据的一律写进「待人工确认」，不要写成「通过」。

## 5. 常见坑

1. **只改 `Info.plist`，但工程用的是自动生成 Info.plist**：真实生效的是 `pbxproj` 里的 `INFOPLIST_KEY_*`，改错位置等于没改。
2. **`PrivacyInfo.xcprivacy` 放在磁盘上没加进 target**：Xcode 打包不包含它，上传或审核阶段才会暴露。
3. **Required Reason API 只声明类别不填 reason**：仍然不合格；reason 代码必须查官方文档。
4. **`NS*UsageDescription` 写「需要此权限才能使用 App」**：这类空话会被问用途；要写清具体做什么、数据去哪。
5. **权限 key 写了但没用**：会被当成索取无关数据（5.1.1/2.3）。
6. **付费墙只有购买按钮、没有恢复购买**：3.1.1 高频拒审点。
7. **订阅页价格写死在代码/截图里**：应当用 StoreKit 返回的价格与周期，否则区域定价不一致会被判误导（2.3/3.1.2）。
8. **站外支付（微信/支付宝/网页下单）解锁数字内容**：3.1.1 直接拒，别指望「只是给个入口」。
9. **删号只提供邮件或电话**：5.1.1 要求 App 内可发起删除。
10. **demo 账号密码过期或后端审核期间不可用**：表现为 2.1 拒审，实际是运维问题。
11. **截图用设计稿或占位图**：2.3 元数据的经典拒审点，截图必须是真实运行界面。
12. **模板残留 `com.example`、`TEMPLATE`**：既是 4.3 重复 App 的风险信号，也是 2.1 观感问题。
13. **隐私政策只在网页上有、App 内点不到**：ASC 有 URL 不等于满足 App 内可达要求。
14. **只信一次 Checklist 就提交**：App Store 政策与上传要求会变（SDK、隐私清单、备案），每次提审前重跑本技能。

## 6. 请核对最新版

本技能基于常见拒审模式总结，**不逐字引用 Apple 准则原文**，条款编号对应的正式表述、当前 SDK/最低系统版本要求、第三方 SDK 清单、隐私清单与 Required Reason API 的类别与 reason 代码，**都可能已经更新**。

提审前请核对（以官方最新版为准）：

- App Review Guidelines（含 2.1 / 2.3 / 3.1.1 / 3.1.2 / 4.2 / 4.3 / 5.1.1 各节）
- App Store Connect 帮助：App 信息、App 隐私、订阅与内购、App 审核信息各页面的字段说明
- Required Reason API 与隐私清单的官方说明
- 上传与 SDK 要求页面（Submitting apps to the App Store）
- 中国区备案：工信部 APP 备案要求与 ASC 备案号字段（第三方实践文章仅供参考，不作依据）

细节查法与修复步骤见 `references/guidelines-cheatsheet.md`。
