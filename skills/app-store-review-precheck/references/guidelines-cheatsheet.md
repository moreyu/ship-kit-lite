# 常见拒审点速查表（guidelines-cheatsheet）

> 用途：`app-store-review-precheck` 技能的执行细则。每条给「症状 → 代码里怎么查（rg 命令）→ 修复方法」。
> 口径：只写条款编号 + 已确认的要点，**不逐字引用 Apple 准则原文**。所有条款表述、SDK 清单、reason 代码请以 Apple 官方最新版为准。
> 扫描前置：在 Xcode 工程根目录执行；统一排除生成目录。

```bash
# 所有命令建议的统一形态（先确认工程规模，必要时去掉排除项）
rg --files -g '!Pods/**' -g '!build/**' -g '!DerivedData/**' | head -200
```

---

## 0. 一键体检（先跑这组，抓大放小）

```bash
# 权限用途字符串声明情况
rg -o 'NS[A-Za-z]+UsageDescription' -g '**/*.plist' -g '*.pbxproj' | sort | uniq -c
# 隐私清单是否存在 / 是否进 target
rg --files -g 'PrivacyInfo.xcprivacy'
rg -n 'PrivacyInfo' *.xcodeproj/project.pbxproj
# 账号体系 / 删号
rg -ni 'ASAuthorization|signInWithApple|FirebaseAuth|Supabase|Auth0|Cognito|signIn\(' -g '*.swift'
rg -ni 'deleteAccount|delete_account|deleteUser|注销账号|删除账号' -g '!Pods/**'
# 内购 / 恢复购买 / 外链支付
rg -n 'Product\.products|\.purchase\(|Transaction\.currentEntitlements|AppStore\.sync' -g '*.swift'
rg -ni 'restore' -g '*.swift'
rg -ni 'alipay|wechatpay|weixin|stripe|paypal|checkout|扫码支付|激活码|兑换码' -g '!Pods/**'
# 套壳
rg -n 'WKWebView|SFSafariViewController|loadRequest|loadHTMLString' -g '*.swift'
# 未完成痕迹 / 模板残留
rg -ni 'TODO|FIXME|Lorem ipsum|coming soon|敬请期待|placeholder|测试版|beta' -g '!Pods/**'
rg -ni 'com\.example\.|TEMPLATE|Copy of' -g '!Pods/**'
# 崩溃高风险
rg -n '\btry!|\bas!\b|fatalError|preconditionFailure' -g '*.swift'
```

---

## 1. Guideline 2.1 — App 完整性（崩溃 / 测试账号 / 审核备注）

**症状**

- 审核期间崩溃、白屏、卡在加载；断网或后端挂掉后完全不可用。
- 需要登录才能看到核心功能，但审核备注里没给 demo 账号，或给的账号在审核期间过期/被限流。
- 存在未完成界面（TODO、敬请期待、假数据占位）。
- 依赖特殊硬件、地区或时间窗口，但备注里没说明。

**代码里怎么查**

```bash
rg -n '\btry!|\bas!\b|fatalError|preconditionFailure|fatalError\(' -g '*.swift'
rg -n '\.(first|last|max|min)!' -g '*.swift'          # 强制解包
rg -ni 'TODO|FIXME|Lorem ipsum|coming soon|敬请期待|mock|dummy' -g '!Pods/**'
rg -n 'URLSession|baseURL\s*=|https?://[a-z0-9.-]+' -g '*.swift' | head -40   # 后端依赖面
rg -ni 'demo|test account|guest|游客' -g '*.swift' -g '*.plist'
```

另外**本地实跑**：Release 构建 + 真机走一遍「冷启动 → 核心功能 → 付费墙 → 断网 → 权限拒绝」，记录任何崩溃与死路。

**修复方法**

1. 把强制解包换成 `guard let` / `if let`；网络与解析错误一律有可读的空态与重试按钮。
2. 后端准备可用性兜底：审核期间保持在线；确有必要时提供本地示例数据，但不得伪造核心功能。
3. 审核备注里写清：demo 账号（长期有效，不看短信/不用二次验证）、订阅测试路径、硬件/地区限制、必须联网的说明。
4. 删掉所有占位文案与假入口。

---

## 2. Guideline 2.3 — 元数据准确

**症状**

- 名称/子标题/描述/截图与真实功能不符，或承诺了没做的功能。
- 截图是设计稿、启动页、登录页，或露出其他平台设备。
- 描述里引导去其他平台下载、写价格承诺或绝对化用语。
- App 隐私问卷与实际采集不一致。

**代码里怎么查（配合 ASC 页面人工核对）**

```bash
rg -ni '第一|最好|#1|永久免费|免费试用|破解|decrypt|无限下载|最准' -g '!Pods/**'
rg -ni 'android|google play|网页版|公众号|加微信' -g '!Pods/**'
rg -n 'displayName|CFBundleDisplayName|CFBundleName' -g '*.plist' -g '*.pbxproj'
rg -ni 'analytics|track|collect|firebase|amplitude|appsflyer' -g 'Package.resolved' -g 'Podfile.lock'
```

**修复方法**

- ASC 逐页核对：**App 信息**（名称 2–30 字符、副标题 ≤30 字符、类别、隐私政策网址）、**版本页**（描述 ≤4000 字符、关键词 ≤100 **字节**（一个汉字 3 字节）、推广文本 ≤170 字符、支持 URL、营销 URL、截图必须是真实界面）、**App 隐私**（数据类型与三方 SDK 一致）。
- 名称/子标题/截图里不出现「beta」「测试版」。
- 描述只写已实现功能，去掉与其他平台相关的引导与价格承诺。

---

## 3. Guideline 3.1.1 — 数字内容必须用内购（含恢复购买）

**症状**

- App 内解锁数字内容/功能/去广告，却给了微信、支付宝、二维码、网页下单、激活码兑换入口。
- 有购买按钮但没有恢复购买；重装后已购内容丢失且无法找回。
- 内购产品 ID 与 ASC 不一致、价格与 ASC 不一致。

**代码里怎么查**

```bash
rg -ni 'alipay|wechatpay|weixin|wxpay|stripe|paypal|checkout|二维码|扫码支付|激活码|兑换码|请到网页购买|购买链接' -g '!Pods/**'
rg -n 'Product\.products|\.purchase\(|Transaction\.currentEntitlements|AppStore\.sync|Product\.PurchaseResult' -g '*.swift'
rg -ni 'restore|恢复购买' -g '*.swift'
rg --files -g '*.storekit'
rg -n 'productIdentifier|com\.[a-z0-9.]+\.(pro|premium|lifetime|monthly|yearly)' -g '*.swift'
```

再对照 ASC →「我的 App」→「订阅」/「App 内购买项目」：产品 ID、类型、价格、本地化、审核截图、状态。

**修复方法**

1. 所有数字内容的解锁一律走 StoreKit（IAP），删除站外支付入口与话术。
2. 在付费墙与设置页都放「恢复购买」，实现 `AppStore.sync()` 或遍历 `Transaction.currentEntitlements` 交付权益。
3. 产品 ID 与 ASC 严格一致；用 `product.displayPrice` 展示价格，不写死金额。
4. 实物商品、App 外服务才可考虑第三方支付；不要拿「外部购买链接」之类的地区特例当通用做法。

---

## 4. Guideline 3.1.2 — 订阅信息披露

**症状**

- 订阅页只说「免费试用 / 订阅」，不写价格、周期、自动续订、如何取消。
- 试用结束后自动扣费的条件下不在付费前展示。
- 没有付费条款（EULA）与隐私政策链接，或链接 404。
- 没有恢复购买。

**代码里怎么查**

```bash
rg -n 'subscriptionPeriod|displayPrice|introductoryOffer|isEligibleForIntroOffer|renewalInfo' -g '*.swift'
rg -ni 'terms|eula|privacy|隐私政策|用户协议|服务条款' -g '*.swift'
rg -ni 'auto-?renew|自动续订|随时取消|试用' -g '*.swift'
```

**修复方法**

付费墙在购买按钮附近必须同时呈现：

1. 价格与计费周期（来自 StoreKit，而非硬编码）；
2. 「到期自动续订，可在系统设置 → Apple 账户 → 订阅 中随时取消」；
3. 免费试用/优惠结束后的价格与扣费条件；
4. 付费条款与隐私政策链接（可点开、非 404）；
5. 恢复购买入口。

---

## 5. Guideline 4.2 — 最低功能（纯 WebView 套壳风险）

**症状**

- 主界面就是一个 `WKWebView` 加载自家网站，原生代码只有登录、设置、壳。
- 内容全部靠远端 HTML 下发；离线、无通知、无系统能力集成。
- 和网页版体验完全一致，没有「只有 App 才能做」的东西。

**代码里怎么查**

```bash
rg -n 'WKWebView|WKWebViewConfiguration|loadRequest|loadHTMLString|SFSafariViewController' -g '*.swift'
rg -n 'WKNavigationDelegate|didFinish|decidePolicyFor' -g '*.swift'
rg -c 'WKWebView' -g '*.swift'                # 出现次数，判断是不是主界面
rg -n 'UNUserNotificationCenter|WidgetKit|CoreLocation|AVFoundation|HealthKit|CloudKit|NSManagedObject|CoreData|SwiftData' -g '*.swift'
```

**修复方法**

- 增加原生价值并写进备注：离线缓存、推送提醒、相机/定位/健康数据、Widget、快捷指令、Apple Watch、iCloud 同步、本地数据模型与检索。
- 将核心流程从 WebView 迁到原生视图；WebView 只保留帮助/条款等非核心页。
- 审核备注说明 App 相对网站提供的额外功能。

---

## 6. Guideline 4.3 — 重复 App（Spam）

**症状**

- 模板换皮：同一份代码改名字、图标、配色多次提交。
- 与开发者账号下已有 App、或商店里成熟 App 高度相似且无实质差异。
- 工程里残留 `com.example`、`TEMPLATE`、示例文案。

**代码里怎么查**

```bash
rg -ni 'com\.example\.|TEMPLATE|MyApp|SampleApp|Copy of' -g '!Pods/**' -g '!*.lock'
rg -n 'PRODUCT_BUNDLE_IDENTIFIER' *.xcodeproj/project.pbxproj
rg -ni 'productIdentifier|productIdentifierPrefix' -g '*.swift' -g '*.plist'
```

**修复方法**

- 清理模板残留（Bundle ID、产品 ID、常量、示例字符串）。
- 保证与同账号其他 App 在功能/内容/交互上有实质区别；不要靠一键换皮批量提交。
- 名称与关键词不蹭其他 App 品牌词（同时也规避商标风险）。

---

## 7. Guideline 5.1.1 — 隐私政策 / 账号删除 / 权限用途字符串

### 7.1 隐私政策链接

**症状**：ASC 没填隐私政策 URL，或填了但 App 内找不到入口；URL 404 或需要登录才能看。

```bash
rg -ni 'privacyPolicy|privacy_policy|隐私政策|privacy\.html|/privacy' -g '*.swift' -g '*.plist' -g '!Pods/**'
```

**修复**：ASC「App 信息」→「隐私政策网址」填公开 https 页面；App 内设置页也放可点链接；两处 URL 一致。

### 7.2 App 内账号删除

**症状**：App 支持注册/登录，但没有任何删除账号入口，或只让用户发邮件/打电话。

```bash
rg -ni 'ASAuthorization|signInWithApple|FirebaseAuth|Supabase|Auth0|Cognito|LoginView|signIn\(|register\(' -g '*.swift'
rg -ni 'deleteAccount|delete_account|deleteUser|注销账号|删除账号|delete account' -g '!Pods/**'
rg -ni 'mailto:|联系我们|客服邮箱' -g '*.swift'        # 仅邮件=不合格信号
rg -n 'UserDefaults.*removeObject|SecItemDelete|keychain|clearAll|cache\.removeAll' -g '*.swift'
```

**修复**：在设置/账户页加「删除账号」入口，二次确认后调用后端删除接口，并清理本地数据、Keychain、缓存与订阅引导（提示需在系统订阅管理里自行取消）。删除必须无需联系客服即可完成。

### 7.3 权限用途字符串（NS*UsageDescription）

**症状**：用到系统能力却没写用途字符串 → 运行时闪退（2.1）；写了但没用 → 被问索取无关权限（5.1.1/2.3）；文案空洞（「需要此权限才能使用 App」）。

```bash
rg -o 'NS[A-Za-z]+UsageDescription' -g '**/*.plist' -g '*.pbxproj' | sort | uniq -c
rg -n 'INFOPLIST_KEY_NS[A-Za-z]+UsageDescription|GENERATE_INFOPLIST_FILE' -g '*.pbxproj'
rg -n '<key>NS[A-Za-z]+UsageDescription</key>' -A1 -g '**/Info.plist'
# 反向：API 是否真的被调用
rg -n 'AVCaptureDevice|PHPickerViewController|PHPhotoLibrary|CLLocationManager|CNContactStore|CMMotion|SFSpeechRecognizer|CBPeripheralManager|LAContext|ATTrackingManager|NWBrowser|NSBonjourServices' -g '*.swift'
```

常见键：`NSCameraUsageDescription`、`NSPhotoLibraryUsageDescription`、`NSPhotoLibraryAddUsageDescription`、`NSMicrophoneUsageDescription`、`NSLocationWhenInUseUsageDescription`、`NSLocationAlwaysAndWhenInUseUsageDescription`、`NSContactsUsageDescription`、`NSMotionUsageDescription`、`NSSpeechRecognitionUsageDescription`、`NSBluetoothAlwaysUsageDescription`、`NSCalendarsFullAccessUsageDescription`、`NSRemindersFullAccessUsageDescription`、`NSHealthShareUsageDescription`、`NSFaceIDUsageDescription`、`NSUserTrackingUsageDescription`、`NSLocalNetworkUsageDescription`。

**修复**：key 与真实 API 一一对应（缺的补、多余的删），文案写「具体做什么 + 数据去哪」，例：「相机用于扫描收据并仅在设备本地识别」。若工程用自动生成 Info.plist，改 `pbxproj` 的 `INFOPLIST_KEY_*`。

---

## 8. PrivacyInfo.xcprivacy 与 Required Reason API

**症状**

- 工程里没有 `PrivacyInfo.xcprivacy`，或文件存在但没加进 target（打包不包含）。
- 用了 Required Reason API（`UserDefaults`、系统启动时间、文件时间戳、磁盘空间、活跃键盘）却没声明类别，或声明了类别没填 reason 代码。
- 上传后被 Apple 邮件以 ITMS 编号（如隐私清单相关的一类）拦回。
- manifest 与 ASC「App 隐私」问卷、第三方 SDK 采集面不一致。

**代码里怎么查**

```bash
rg --files -g 'PrivacyInfo.xcprivacy'
rg -n 'PrivacyInfo' *.xcodeproj/project.pbxproj
rg -n 'NSPrivacyAccessedAPITypes|NSPrivacyAccessedAPICategory|NSPrivacyCollectedDataTypes|NSPrivacyTracking|NSPrivacyTrackingDomains' -g '*.xcprivacy'
rg -n 'UserDefaults|NSUserDefaults' -g '*.swift'
rg -n 'systemUptime|mach_absolute_time|clock_gettime' -g '*.swift'
rg -n 'creationDate|contentModificationDate|attributesOfItem|fileModificationDate' -g '*.swift'
rg -n 'volumeAvailableCapacity|NSFileSystemFreeSize|statfs|statvfs' -g '*.swift'
rg -n 'activeInputModes' -g '*.swift'
```

**类别键对照（Required Reason API）**

| 命中的 API | PrivacyInfo 类别键 | 说明 |
|---|---|---|
| `UserDefaults` / `NSUserDefaults` | `NSPrivacyAccessedAPICategoryUserDefaults` | App 自身设置与状态 |
| `systemUptime` / `mach_absolute_time` | `NSPrivacyAccessedAPICategorySystemBootTime` | 测量 App 内流逝时间 |
| 文件创建/修改时间、`attributesOfItem` | `NSPrivacyAccessedAPICategoryFileTimestamp` | 读取容器内文件时间戳 |
| 剩余磁盘空间（`statfs` 等） | `NSPrivacyAccessedAPICategoryDiskSpace` | 写文件前检查空间 |
| `activeInputModes` | `NSPrivacyAccessedAPICategoryActiveKeyboards` | 判断是否启用自定义键盘 |

每个类别还必须填 Apple 定义的 reason 代码（示例：UserDefaults 常填 `CA92.1`）。**示例仅供参考，代码请在 Apple 官方文档或 Xcode 生成的模板里核对，不要凭记忆抄。**

**修复**

1. 没有 manifest 就新建 `PrivacyInfo.xcprivacy`，并确认出现在 target 的 Copy Bundle Resources；
2. 覆盖四块内容：`NSPrivacyTracking`、`NSPrivacyTrackingDomains`、`NSPrivacyCollectedDataTypes`、`NSPrivacyAccessedAPITypes`；
3. 每个命中的 API 类别都要有对应条目 + 准确 reason 代码；
4. 让 manifest、ASC「App 隐私」问卷、第三方 SDK 声明三处一致。

---

## 9. 第三方 SDK

**症状**：SDK 版本过旧、不含隐私清单或未签名分发；SDK 采集的数据没写进 manifest/ASC 问卷；上传或审核阶段被拦。

```bash
rg -n 'Firebase|Adjust|Amplitude|AppsFlyer|FBSDK|FacebookCore|Sentry|OneSignal|RevenueCat|Umeng|Bugly|AdMob|GADMobileAds' Package.resolved Podfile.lock Cartfile.resolved
rg -n 'Analytics|Crashlytics|ATTrackingManager|IDFA|advertisingIdentifier' -g '*.swift'
```

**修复**：使用 Apple「常用第三方 SDK」清单内 SDK 时，升级到**已包含隐私清单并已签名分发**的版本；把它采集的数据类型同步进 manifest 与 ASC「App 隐私」；不再使用的 SDK 直接移除（少一个 SDK 少一处风险）。

---

## 10. 中国区（非 Guidelines，但提审前必查）

- ASC →「App 信息」→ 中国大陆 ICP 备案区域：填写 App 自己的备案号，ICP 编号与元数据须与工信部备案一致（官方）；据第三方实践文章，随下次提交审核生效、页面待确认提示须点掉。
- 据第三方实践文章，2026 年 7 月起无备案号的新 App 在中国区可能不展示、老 App 发新版后也可能消失，且提交审核本身不校验备案号；**该说法为第三方观察，以官方与接入商最新要求为准**。
- 详细材料与流程见 `china-app-filing` 技能（本表不展开）。

---

## 11. 通用复核命令（改完自跑）

```bash
rg -ni 'alipay|wechatpay|stripe|paypal|扫码支付|兑换码' -g '!Pods/**'   # 应为空
rg -ni 'deleteAccount|注销账号' -g '!Pods/**'                          # 有登录则必须有
rg -n 'Transaction\.currentEntitlements|AppStore\.sync' -g '*.swift'   # 有 IAP 则必须有
rg --files -g 'PrivacyInfo.xcprivacy'                                  # 用到 Required Reason API（如 UserDefaults）时必须存在
rg -n 'UserDefaults|systemUptime|creationDate|volumeAvailableCapacity' -g '*.swift'  # 命中即需声明
rg -n '\btry!|\bas!\b|fatalError' -g '*.swift'                          # 崩溃面
```

**请核对最新版**：本速查表为常见拒审模式总结，不逐字引用 Apple 准则原文；条款编号对应的正式表述、Required Reason API 类别与 reason 代码、第三方 SDK 清单、SDK/最低系统版本要求、中国区备案要求均可能已更新。提审前请以 Apple App Review Guidelines、App Store Connect 帮助与工信部最新要求为准。
