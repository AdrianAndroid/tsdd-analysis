# TangSengDaoDaoiOS 深度分析报告

> 分析对象：`/data/code/tsdd-analysis/TangSengDaoDaoiOS`
> 分析方式：纯静态代码审查（未编译、未运行动态测试）
> 分析日期：2026-10-12
> 说明：文中所有 `文件:行号` 引用均经 Read/grep 实际确认；不确定处标注"待验证"。
> 路径约定：`Modules/WuKongBase/...` 均指仓库内本地 pod 目录；为简洁起见部分长路径省略中间层级。

---

## 目录

1. [仓库概述](#1-仓库概述)
2. [技术栈与依赖](#2-技术栈与依赖)
3. [目录结构与模块架构](#3-目录结构与模块架构)
4. [核心流程走读](#4-核心流程走读)
5. [对外接口](#5-对外接口)
6. [部署与运维](#6-部署与运维)
7. [安全审计](#7-安全审计)
8. [代码质量评估](#8-代码质量评估)
9. [风险与改进建议](#9-风险与改进建议)
10. [附录](#附分析覆盖度与遗留待验证项)

---

## 1. 仓库概述

**TangSengDaoDaoiOS 是"唐僧叨叨"（TangSengDaoDao，一款基于 WuKongIM 的开源即时通讯应用）的 iOS 客户端主仓库。** Bundle ID 为 `com.xinbida.tangsengdaodao`（`TangSengDaoDaoiOS.xcodeproj/project.pbxproj:879`），应用内名称"唐僧叨叨"（`Modules/WuKongBase/WuKongBase/Classes/WKAppConfig.m:30`），官方文档站点为 tsdaodao.com（`README.md:3` 仅一行指向 `https://tsdaodao.com/dev/ios/intro.html`）。仓库根目录另有 `appstore.md`、`TangSengDaoDaoCli.md`、`TangSengDaoDaoJSBridge.md` 等关联仓库的说明文件，显示它属于一个多端（Android/iOS/CLI/JSBridge）产品族。

仓库内容构成（排除 `Pods/`）：

| 构成 | 规模 | 说明 |
|---|---|---|
| Objective-C `.m` | 629 个文件 | 主体语言（含 Modules/ 内 vendor 的 IM SDK 源码） |
| Objective-C `.h` | 720 个文件 | |
| Swift `.swift` | 225 个文件 | 其中 ~221 个集中在 WuKongBase（含大量 Telegram 移植工具库） |
| 总代码行数（.m+.swift，不含 Pods/Example/Tests） | 约 15.4 万行 | 含 vendor 代码 |
| `Pods/` | 189MB、1496 个 .h | 整体提交进版本库 |

### 核心取舍：vendor 整个 WuKongIM SDK 源码

本仓库最大的架构特征是**把 WuKongIM iOS SDK 的完整源码以本地 pod 形式 vendor 进仓库**（`Modules/WuKongIMiOSSDK/`，含 coder/db/manager/model 全部实现，podspec 版本 1.1.0，见 `Modules/WuKongIMiOSSDK/WuKongIMSDK.podspec:11`）。Podfile 中同时保留了注释掉的两种替代方案（`Podfile:57-59`）：

```ruby
pod 'WuKongIMSDK',  :path => './Modules/WuKongIMiOSSDK'   ## WuKongBase 基础工具包  源码地址 https://github.com/WuKongIM/WuKongIMiOSSDK
#  pod 'WuKongIMSDK',  :path => '../../../wukongIM/iOS/WuKongIMiOSSDK'
#  pod  'WuKongIMSDK', '~> 1.0.2' ## 源码地址 https://github.com/WuKongIM/WuKongIMiOSSDK
```

对比同生态的 Android 端（TangSengDaoDaoAndroid 通过 JitPack 以二进制 aar 依赖 WuKongIM Android SDK——此为跨仓库背景信息，本仓库内无法直接验证），iOS 端选择源码 vendor 的利弊：

- **利**：可随时修改 SDK 行为（本仓库 SDK 内有 `WKSecurityManager`、`WKRobotManager`、`WKFlameManager`（阅后即焚）等模块，定制自由度高）；不受上游发布节奏约束；调试可断点到协议层；不受上游仓库可用性影响。
- **弊**：与上游 `WuKongIM/WuKongIMiOSSDK` 仓库产生分叉，协议升级与安全修复需要手工回填；`Modules/` 下各 pod 还自带 `Example/`（含自己的 `Pods/` 目录，如 `Modules/WuKongLogin/Example/Pods`）与 stub `Tests/`，仓库体积进一步膨胀；`Pods/`（189MB）整体提交进版本库，使 diff 噪音与克隆成本都很高。

### 仓库内的其他 vendor 行为

除 IM SDK 外，仓库还有三层 vendor：

1. **腾讯 Bugly 静态框架**：`Modules/WuKongBase/WuKongBase/Bugly.framework`（二进制，经 `WuKongBase.podspec:63` `s.vendored_frameworks = 'WuKongBase/Bugly.framework'` 引入，不走 CocoaPods 版本管理）。
2. **WebViewJavascriptBridge**：`Modules/WuKongBase/WuKongBase/Classes/Vendor/WebViewJavascriptBridge/`（5 个文件源码拷贝，与 TangSengDaoDaoJSBridge 仓库呼应，见 §7.4）。
3. **Telegram iOS 客户端的 Swift 工具库**：`Classes/Sections/Common/TelegramUtils/` 下约 18 个子库源码拷贝（Display、SwiftSignalKit、ContextUI、AnimatedStickerNode、RadialStatusNode、ReactionSelectionNode、Markdown、ShimmerEffect 等，合计约 200 个 Swift 文件），用于高性能列表/上下文菜单/动效渲染。这是典型的"抄 Telegram 作业"式集成，体量巨大且与上游无同步机制。

综合来看，本仓库的 vendor 程度在同类开源 IM 项目中属于非常重的：**IM 协议栈、崩溃上报、JS 桥、UI 底层库全部内化**，自包含程度极高，但供应链可维护性很差。

---

## 2. 技术栈与依赖

### 2.1 构建与依赖管理

- **CocoaPods 1.16.2**（`Podfile.lock` 末行 `COCOAPODS: 1.16.2`），无 Swift Package Manager；workspace 为 `TangSengDaoDaoiOS.xcworkspace`。
- 平台：`platform :ios, '12.0'`（`Podfile:2`）；post_install 钩子把所有 Pods target 的部署目标压到 **11.0**、关闭 Bitcode、模拟器排除 arm64（`Podfile:24-32`）。
- post_install 中**硬编码了开发者 Team ID `H8PU463W68`**（`Podfile:6`），用于给 bundle 类型 pod target 补签名。
- `Pods/` 目录整体入库（189MB），新人克隆后无需 `pod install` 即可构建，但依赖升级 diff 巨大。
- Podfile 结构：`abstract_target 'TangSengDaoDaoiOSBase'` 声明公共 pod（Socket.IO-Client-Swift、SSZipArchive、SocketRocket、Aspects、ReactiveObjC），内嵌主 target `TangSengDaoDaoiOS` 再声明 4 个 fork git pod + 5 个本地 path pod，并开启 `use_frameworks!`（`Podfile:36-67`）。

### 2.2 关键依赖表（来自 Podfile.lock）

| 依赖 | 版本 | 用途 | 备注 |
|---|---|---|---|
| AFNetworking | 4.0.1 | REST 网络层（WKAPIClient 封装） | 安全策略被弱化，见 §7.1 |
| PromiseKit/CorePromise | 6.18.1 | 异步链式调用（`.then/.catch`） | 与 AFNetworking 配合，贯穿全部 REST 调用 |
| FMDB/SQLCipher + SQLCipher | 2.7.5 / 4.5.4 | 本地消息库加密 | 密钥强度问题见 §7.2 |
| CocoaAsyncSocket | 7.6.5 | IM 长连接 TCP（GCDAsyncSocket） | 未启用 TLS，见 §7.1 |
| 25519 | 2.0.2 | Curve25519 DH 密钥协商 | WuKongIMSDK 依赖 |
| Socket.IO-Client-Swift + Starscream | 15.2.0 / 3.1.1 | abstract target 声明 | 主代码未定位到实际调用点（待验证，疑为历史遗留） |
| SocketRocket | 0.6.0 | WebSocket | 同上 |
| SSZipArchive | 2.2.3 | 压缩 | 表情包/资源包 |
| ReactiveObjC | 3.1.1 | 响应式 | |
| Aspects | 1.4.1 | AOP hook | **声明但未在自有代码中使用**（全仓 grep `aspect_hook` 仅命中 Pods 自身），死依赖，见 §7.6 |
| AsyncDisplayKit | 1.0.0（fork） | UI 渲染框架（Texture） | 来自 `tangtaoit/AsyncDisplayKit`，配合 TelegramUtils |
| lottie-ios | 2.5.3 | 老版 Lottie（ObjC 时代版本） | 2019 年版本，已非常陈旧 |
| librlottie + SDWebImageLottieCoder | 0.1.7 / 0.1.0 | rlottie 渲染 tgs 贴图 | fork |
| SDWebImage(+WebPCoder) | 5.9.5 / 0.6.1 | 图片加载与 webp | 自定义 `WKSDWebImageDownloaderOperation` |
| Masonry / MJRefresh / MBProgressHUD / Toast | 1.1.0 / 3.7.5 / 1.1.0 / 4.0.0 | UI 基础件 | |
| YBImageBrowser/NOSD | 3.0.9（fork） | 图片浏览 | fork |
| YYImage/WebP | 1.0.4（fork） | 图片编解码 | fork |
| TZImagePickerController / ZLPhotoBrowser / ZLImageEditor / TOCropViewController | 3.6.9 / 4.5.5 / 1.1.7 / 2.5.5 | 图片选择/编辑/裁剪 | |
| LBXScan（LBXNative+LBXZXing+UI） | 2.5.1 | 二维码扫描 | 扫码加好友/扫码登录 |
| CocoaLumberjack | 3.8.0 | 日志框架 | 但业务宏被改写为 NSLog，见 §7.5 |
| M80AttributedLabel / ActionSheetPicker-3.0 / DGActivityIndicatorView / FPSCounter / GZIP / libwebp 1.2.4 | 若干 | 富文本/选择器/加载/帧率/压缩 | FPSCounter 仅 DEBUG 显示（`WKApp.m:250-256`） |
| Bugly | vendored framework（无版本号可查） | 崩溃上报 | 非 CocoaPods 引入，AppID 硬编码，见 §7.6 |

### 2.3 fork 仓库（tangtaoit/*）的维护风险

Podfile 直接以 `:git` 指向作者个人账号 fork 的 4 个仓库，并以 commit 锁定（`Podfile.lock` CHECKOUT OPTIONS 段）：

| fork | 锁定 commit | 上游状况 |
|---|---|---|
| tangtaoit/AsyncDisplayKit | 3f2a0b8f5069ddefd53cf4796da22eec105c5c7c | Texture 已基本停更 |
| tangtaoit/librlottie | bed3cfe62cdb77d728adfe76893ff51b69c05e08 | Samsung rlottie，更新缓慢 |
| tangtaoit/YBImageBrowser | 0663a468dea897acf7dc66e7128b71e3af48aea4 | 原仓库停更 |
| tangtaoit/YYImage | be7dc29bbd79153ea03c5018ee2ab2512a16f3fd | ibireme/YYImage 多年未维护 |

风险：

- 上游多已停更，**图片/动图解析器历来是越界读写类 CVE 高发区**（webp、gif、tgs），fork 停留在私有修改点，安全补丁无法通过升级获得。
- 个人账号仓库若被删除/改名/强制推送，全新环境 `pod install` 直接失败（当前靠 Pods 入库掩盖了这一问题，属于"隐性单点依赖"）。
- fork 内私有修改无文档，后续维护者无法判断与上游的 diff 意图。

### 2.4 iOS 12 部署目标的现代性

- 主 target `IPHONEOS_DEPLOYMENT_TARGET = 12.0`（pbxproj 中共 8 处，含两个扩展 target），Pods 被 post_install 压到 11.0。
- iOS 12（2018 年发布）在 2026 年已非常陈旧；代码中大量 `@available(iOS 13.0, *)` 分支（如 `AppDelegate.m:82-88` 深色模式、`WKMainTabController.m:31-37`、`AppDelegate.m:137` deviceToken 处理），说明实际运行环境按 13+ 设计。停留在 12 的收益（老设备覆盖）与成本（无法用现代 API、Xcode 新版本警告、安全基线）需要重新权衡。
- **更正背景信息**：背景中提及的 WechatOpenSDK 在本仓库并不存在（全仓 grep `WechatOpenSDK`/`WXApi` 无命中，Podfile.lock 亦无）；Bugly 也不是 pod 依赖，而是二进制 framework vendor。

---

## 3. 目录结构与模块架构

### 3.1 顶层结构

```
TangSengDaoDaoiOS/
├── TangSengDaoDao/            # 主 app target（仅 4 个 .m：main/AppDelegate/ViewController/WKMainTabController）
│   ├── AppDelegate.m          # 启动装配 + 硬编码服务器 + 版本检查 + deviceToken 上报（180 行）
│   ├── WKMainTabController.m  # 三个 tab 容器（73 行）
│   ├── ViewController.m       # 模板遗留，未使用
│   ├── Info.plist             # ATS 豁免、botgate scheme
│   ├── TangSengDaoDao.entitlements  # aps-environment=development、applinks:githubim.com
│   └── zh-Hans.lproj / en.lproj     # 主 app 文案（几乎为空，文案在 WuKongBase）
├── NotificationService/       # APNs 通知服务扩展 target（stub 实现，36 行）
├── NotificationContent/       # APNs 富通知 UI 扩展 target（模板代码）
├── Modules/                   # 5 个本地 pod（各含 podspec/Example/stub Tests）
│   ├── WuKongIMiOSSDK/        # WuKongIM SDK 完整源码（coder/db/manager/model/utils/private）
│   ├── WuKongBase/            # 应用框架层：WKApp 单例 + 全部功能 Sections + Services + Vendor + Bugly.framework
│   ├── WuKongLogin/           # 登录/注册/找回密码/扫码授权登录/授权 WebView
│   ├── WuKongContacts/        # 通讯录 UI 与好友逻辑
│   └── WuKongDataSource/      # WuKongBase 各 Manager 的 delegate 实现（数据装配层）
├── Vendor/UpdateAlert/        # 版本更新弹窗（SELUpdateAlert）
├── TangSengDaoDao-Bridging-Header.h  # 内容几乎为空（仅注释）
├── Podfile / Podfile.lock
└── Pods/                      # 189MB，已提交
```

三个 Xcode target（pbxproj `PRODUCT_BUNDLE_IDENTIFIER` 段）：

| target | Bundle ID | 代码量 |
|---|---|---|
| TangSengDaoDaoiOS | com.xinbida.tangsengdaodao | 4 个 .m（pbxproj Sources 段确认） |
| NotificationService | com.xinbida.tangsengdaodao.NotificationService | 1 个 .m |
| NotificationContent | com.xinbida.tangsengdaodao.NotificationContent | 1 个 .m + storyboard |

### 3.2 target / 本地 pod 关系图

```mermaid
graph TD
    subgraph Xcode Workspace
        APP["TangSengDaoDaoiOS<br/>(主 app, 4 个 .m)"]
        NSE["NotificationService<br/>(APNs 服务扩展, stub)"]
        NCE["NotificationContent<br/>(APNs 富通知 UI, 模板)"]
    end

    subgraph 本地 Pods（Modules/）
        LOGIN["WuKongLogin<br/>登录/注册/授权登录"]
        CONTACTS["WuKongContacts<br/>联系人/好友"]
        DS["WuKongDataSource<br/>数据源装配(delegate 实现)"]
        BASE["WuKongBase<br/>框架层 WKApp + 功能 Sections + Services"]
        IMSDK["WuKongIMSDK<br/>IM 协议栈(vendored 源码)"]
    end

    APP -->|path pod| LOGIN
    APP -->|path pod| CONTACTS
    APP -->|path pod| DS
    APP -->|path pod| BASE
    LOGIN --> BASE
    CONTACTS --> BASE
    DS --> BASE
    BASE --> IMSDK
    CONTACTS --> IMSDK
    BASE -->|vendored_frameworks| BUGLY["Bugly.framework"]
    BASE -->|源码 vendor| WVJB["WebViewJavascriptBridge"]
    BASE -->|源码 vendor| TG["TelegramUtils (Swift, ~18 子库)"]
    IMSDK --> SQLC["FMDB/SQLCipher"]
    IMSDK --> SOCK["CocoaAsyncSocket"]
    IMSDK --> C25519["25519 (Curve25519)"]
    BASE --> AFN["AFNetworking + PromiseKit"]
    NSE -. 无 pod 依赖 .-> APP
    NCE -. 无 pod 依赖 .-> APP
```

注：NotificationService / NotificationContent 两个扩展 target 不链接任何 pod（源码中无三方 import），与主 app 无代码共享、无 App Group 数据交互痕迹（entitlements 中未见 app groups，待验证是否与主 app 共享容器——从 stub 实现看没有）。

### 3.3 WKApp 单例框架机制

整个 app 的业务逻辑几乎全部生活在 WuKongBase 的 `WKApp` 单例中（`Modules/WuKongBase/WuKongBase/Classes/WKApp.m`，1782 行）：

- **单例**：`+ (WKApp *)shared`（`WKApp.m:133-141`），`allocWithZone:` 同样做了防重入（`WKApp.m:125-132`）。
- **配置注入**：宿主 app 在启动时构造 `WKAppConfig` 并赋值 `[WKApp shared].config`（`AppDelegate.m:62-70`）；`config` 惰性初始化（`WKApp.m:143-148`）。`WKAppConfig` 承载全部主题/尺寸/文案/URL 配置，默认值在 `WKAppConfig.m:24-96` 的 `init` 中（appName "唐僧叨叨"、`appSchemaPrefix = @"wukong"`、`clusterOn = YES`、主题色 `rgb(228,99,66)`、系统 uid `u_10000`、文件助手 `fileHelper` 等）。
- **首页 block 注入**：宿主通过 `getHomeViewController` block 提供首页 VC（声明于 `WKApp.h:68`；赋值于 `AppDelegate.m:73-76`），登录成功或已登录时由 `[[WKNavigationManager shared] resetRootViewController:]` 切换（`WKApp.m:357-359`、`WKApp.m:401-403`）。
- **appInit 总装**：`-(void)appInit`（`WKApp.m:274-441`）依次完成：
  1. `configApi`——WKAPIClient baseURL/公共头/路径替换/统一错误处理（`WKApp.m:277` → `WKApp.m:524-579`）；
  2. 语言适配 ZLPhotoBrowser（`WKApp.m:279-283`）；
  3. `WKKeyboardService` 与 `WKSwiftModuleManager didModuleInit`（`WKApp.m:285-286`）；
  4. `traceConfig`——Bugly 初始化（`WKApp.m:291` → `WKApp.m:234-248`）；
  5. SDWebImage coder 注册（webp/lottie，`WKApp.m:199-212`）；
  6. 前后台/录屏/内存通知监听（`WKApp.m:160-177`）；
  7. 网络监听 `WKNetworkListener`、日志 `WKLogsManager setup:nil`（`WKApp.m:298-302`）；
  8. 登录信息加载、系统消息处理器（`WKApp.m:305-309`）；
  9. point 方法注册（`initPointMethods`）、消息类型注册（`registerMessages`，`WKApp.m:214-232`：文本/图片/语音/GIF/名片/输入中/合并转发/历史分割线/E2E 提示/解密失败/截屏通知/lottie 贴图/emoji 贴图等 content→cell 映射）；
  10. IM SDK 配置：`connectURL`（`WKApp.m:317`）、每页同步条数（`WKApp.m:319`）、连接信息回调（uid+imToken，`WKApp.m:323-328`）、集群取地址（`WKApp.m:332-346`）；
  11. 登录/登出 point 处理器（`WKApp.m:349-396`）；
  12. 登录态分支（`WKApp.m:398-410`）。
- **Endpoint/Point 机制（自研服务定位）**：`WKApp` 暴露 `invoke:param:` / `invokes:category:param:` / `setMethod:handler:` 等（`WKApp.h:136-147`），底层是 `WKEndpointManager`（`Services/Core/WKEndpointManager.h:13`）。所有跨模块调用都通过字符串 point 解耦，point 常量在 `WKConstant.h` 定义，例如：
  - `WKPOINT_LOGIN_SHOW = @"login.show"`（`WKConstant.h:19`）
  - `WKPOINT_LOGIN_SUCCESS = @"login.success"`（`WKConstant.h:21`）
  - `WKPOINT_LOGIN_LOGOUT = @"login.logout"`（`WKConstant.h:25`）
  - `WKPOINT_CONVERSATION_SHOW`、`WKPOINT_CATEGORY_PANEL`（输入面板扩展点）、`WKPOINT_CATEGORY_MESSAGE_LONGMENUS`（长按菜单扩展点）等（`WKConstant.h:19-89`）。

  模块类用 `@WKModule(...)` 宏注册（ObjC 侧，`WKModuleManager`）或由 `WKSwiftModuleManager`（`Services/Core/WKSwiftModuleManager.swift`）驱动 Swift 模块生命周期（didModuleInit/didFinishLaunching/didOpenURL/didReceiveRemoteNotification 等）。例如 WuKongLogin 在 `moduleInit` 里注册 `login.show` 处理器弹出 `WKLoginVC`（`Modules/WuKongLogin/WuKongLogin/Classes/Login/WKLoginModule.m:26-32`），并注册扫码处理器 `WKPOINT_SCAN_HANDLER_GRANTLOGIN` 处理 `loginConfirm` 类型扫码（`WKLoginModule.m:35-47`）。

这种"字符串 point + category"的服务定位机制相当于手写了一个弱类型 EventBus/DI 容器：解耦彻底（WuKongBase 不 import WuKongLogin 的任何头文件即可弹登录页），但 point 拼写错误、参数协议都没有编译期检查，重构成本高。

### 3.4 WuKongBase Sections 功能划分表

`Modules/WuKongBase/WuKongBase/Classes/Sections/` 下按业务分 11 个 Section：

| Section | 职责 | 代表文件（行数） |
|---|---|---|
| ConversationList | 最近会话列表 | `WKConversationListVC.m`（863）、`WKConversationListVM` |
| Conversation | 聊天页（消息列表、输入面板、机器人、合并转发详情） | `WKConversationVC.m`、`WKConversationView.m`（853）、`WKConversationContextImpl.m`（846）、`WKMessageListView.m`（1599）、`Input/WKConversationInputPanel.m`（1050） |
| Messages | 消息 cell 基类与各种消息类型 cell/content | `WKMessageCell.m`（1164）、`WKTextMessageCell`、`WKImageMessageCell`、`WKVoiceMessageCell` 等 |
| ConversationSetting | 单聊/群聊设置页 | `WKConversationPersonSettingVC`、`WKConversationGroupSettingVC` |
| Contacts | 会话内联系人选择等 | `WKContactsSelectVC` |
| Group | 群资料/群管理 UI | |
| Me | "我的" tab、设置、锁屏、收藏表情管理 | `WKMeVC.m`（被 `AppDelegate.m:14` import）、`WKScreenPasswordVC`、`WKCommonSettingVC` |
| Channel | 频道资料、订阅者 | |
| Scan | 扫码（LBXScan 封装）、扫码登录授权处理 | `WKScanVC`、`WKScanHandler` |
| InlineQuery | 机器人 inline 查询 | |
| Common | 公共组件：Webview（含 JS 桥）、NavigationManager、ActionSheet、用户资料页、**TelegramUtils（vendored Swift 库）** | `WKWebViewVC.m`（497）、`WKNavigationManager.m`、`NSAttributedString+Markdown.m`（1160） |

Services 层（`Classes/Services/`）：

| 子目录 | 内容 |
|---|---|
| Base | `WKAPIClient`（REST）、`WKLoginInfo`（登录态）、`WKModel`（字典→模型） |
| Core | `WKModuleManager`（ObjC 模块）、`WKSwiftModuleManager.swift`、`WKEndpointManager` |
| DB | `WKKitDB`（WuKongBase 侧库，前缀 `wukongkit_`，`WKKitDB.m:19`；`switchDB` 内部联动 `WKDB sharedDB`，`WKKitDB.m:66-73`） |
| Logs | `WKLogs`（CocoaLumberjack 封装） |
| Service | `WKSyncService`（数据同步编排）、`WKSchemaManager`、`WKProhibitwordsService`（违禁词） |
| Category | `UIDevice+Utils`、`NSString+WKLocalized` 等 |

---

## 4. 核心流程走读

### 4.a 启动与配置

调用链：

```
main.m → UIApplicationMain(AppDelegate)                           (main.m:12-15)
AppDelegate.application:didFinishLaunchingWithOptions:            (AppDelegate.m:52)
  ├─ 自建 UIWindow 并 makeKeyAndVisible                           (AppDelegate.m:54-56)
  ├─ [WKApp shared].loginInfo load                                (AppDelegate.m:59 → WKLoginInfo.m:123-128)
  ├─ WKAppConfig 组装（硬编码服务器）                              (AppDelegate.m:62-70)
  │     apiBaseUrl/fileBaseUrl/fileBrowseUrl/imageBrowseUrl = https://api.botgate.cn/v1/
  │     reportUrl / privacyAgreementUrl / userAgreementUrl
  ├─ getHomeViewController = ^{ return [WKMainTabController new]; } (AppDelegate.m:73-76)
  ├─ [WKApp shared] appInit                                       (AppDelegate.m:80 → WKApp.m:274)
  │     ├─ ...（见 §3.3 的 12 步总装）
  │     ├─ isLogined ?                                            (WKApp.m:398；判定逻辑 WKApp.m:602-605：token 非空)
  │     │    ├─ 是：WKKitDB switchDB:uid (WKApp.m:400)
  │     │    │       → resetRootViewController(getHomeViewController()) (WKApp.m:401-403)
  │     │    │       → WKSyncService sync (WKApp.m:405)
  │     │    │       → registerForNotification + connectionManager connect (WKApp.m:419-420)
  │     │    └─ 否：invoke WKPOINT_LOGIN_SHOW → WKLoginVC          (WKApp.m:409 → WKLoginModule.m:26-32)
  │     └─ remoteConfig / 收藏表情加载 / 断网屏幕保护 / 锁屏保护     (WKApp.m:426-435)
  └─ iOS13+ 深色模式应用                                          (AppDelegate.m:82-88)
```

关键事实：

- 服务器地址**硬编码**：`#define SERVER_IP @"api.botgate.cn"`、`#define HTTPS_ON true`（`AppDelegate.m:19-20`），由此拼出 `https://api.botgate.cn/v1/` 作为 API/文件/图片基址（`AppDelegate.m:23-32`），`/web/` 作为 H5 基址（`AppDelegate.m:24`）。
- `WKMainTabController.viewDidLoad` 装三个 tab：`WKConversationListVC`（会话）、`WKContactsVC`（通讯录，来自 WuKongContacts）、`WKMeVC`（我的，来自 WuKongBase Me Section）（`WKMainTabController.m:39-41`）；tab 切换带触感反馈（`WKMainTabController.m:64-71`）。
- 进入前台每 30 分钟检查一次版本：`GET common/appversion/iOS/{版本}`（`AppDelegate.m:93-99` 节流，`AppDelegate.m:106-126` 请求与弹窗），有新版本弹 `SELUpdateAlert`（`Vendor/UpdateAlert/SELUpdateAlert.m`，支持强制更新）。
- 启动后 `enterApp` 还会 `GET user/devices/{uuid}` 拿设备数字 id 注入 SDK（`WKApp.m:444-451`）。

### 4.b 登录与 token

调用链：

```
WKLoginVC（手机号+密码 UI；另有 WKThirdLoginVC 第三方授权、WKLoginPhoneCheckVC 短信登录，可选）
  → WKLoginVM.login:password:                                     (WuKongLogin/Classes/Login/Src/WKLoginVM.m:47-51)
      POST {apiBaseUrl}user/login
      body: {username, password(明文), device:{device_id, device_name, device_model}}
      device_id = [UIDevice getUUID]（IDFV→Keychain，UIDevice+Utils.m:16-33）
  → 成功后 WKLoginVC 内分支：                                     (WuKongLogin/Classes/Login/Src/WKLoginVC.m:56-61)
      WKLoginVM.handleLoginData:isSave:                           (WKLoginVM.m:53-94)
        → WKLoginInfo.token / imToken / uid / name / zone / phone / sex
          / server_id / chat_pwd / lock_screen_pwd / rsa_public_key / setting / avatar 全部写入 loginInfo.extra
        → [loginInfo save]
      → [WKApp invoke:WKPOINT_LOGIN_SUCCESS]                      (WKLoginVC.m:61 → WKApp.m:349-380)
        → WKKitDB switchDB:uid                                    (WKApp.m:351)
        → WKConversationPositionManager reload                    (WKApp.m:354)
        → resetRootViewController(首页 tab)                        (WKApp.m:357-359)
        → WKSyncService sync → 完成后 connectionManager connect    (WKApp.m:361-367)
        → registerForNotification（APNs 授权+注册）                (WKApp.m:369 → WKApp.m:453-520)
        → WKSecurityTipManager syncIfNeed（敏感词）                (WKApp.m:375)
```

token 的存储与使用（安全相关，详见 §7.2/§7.5）：

- **存储位置：NSUserDefaults**。`WKLoginInfo` 实现 `NSCoding`（`WKLoginInfo.m:50-78`），`save` 用 `NSKeyedArchiver` 归档后写入 `NSUserDefaults` 键 `WKLoginInfo`（`WKLoginInfo.m:131-135`），`load` 反归档（`WKLoginInfo.m:123-128`）。**未使用 Keychain**。uid、token、imToken、deviceToken、extra（含手机号、锁屏密码等）全部明文落地于 app 沙盒 plist。
- **注入方式**：每次请求前 `resetPublicHeader` 执行公共头 block，把 token 放进 HTTP header `token`、bundleID 放进 `bundle_id`（block 定义于 `WKApp.m:528-537`，执行于 `WKAPIClient.m:365-375`）；路径中 `{uid}` 占位符替换（`WKApp.m:539-545`）。
- **401 处理**：统一 errorHandler 遇到 401 且已登录 → `immediatelyLogout`（`WKApp.m:547-553`）→ 清登录信息（`WKLoginInfo.m:147-163`：保留区号/手机号/主题设置）→ invoke `WKPOINT_LOGIN_LOGOUT` → 断 IM 连接 + 弹登录页（`WKApp.m:384-396`）。
- 正常登出先 `DELETE user/device_token` 再清本地（`WKApp.m:607-616`）。
- IM 长连接的 token（`imToken`，登录响应 `im_token` 字段，`WKLoginVM.m:21,55-59`）通过 `connectInfoCallback` 提供给 SDK（`WKApp.m:323-328`）。

### 4.c 消息收发（IM SDK 为本地源码，可深入到协议层）

**发送路径**：

```
WKConversationInputPanel（输入框，Input/WKConversationInputPanel.m）
  → delegate: WKConversationView inputPanel:sendMessage:      (Sections/Conversation/WKConversationView.m:683-685)
  → WKConversationContextImpl sendMessage:                    (Sections/Conversation/WKConversationContextImpl.m:495-520)
      （组装 WKSetting、topic，发送后立即把 message 塞进 messageListView 上屏，乐观渲染 :520）
  → WKSDK.chatManager sendMessage:channel:setting:topic:      (IMSDK manager/WKChatManager.m:59-96 重载族)
      → saveMessage 落库（状态 WAITSEND）
  → WKChatManager sendMessage:addRetryQueue:                  (WKChatManager.m:284-321)
      ├─ WKRetryManager add（失败重试队列；重试 8 次/间隔 10s，WKOptions.m:20-21）
      └─ WKMessageQueueManager sendMessage:                   (WKMessageQueueManager.m:49)
            限速：每 sendFrequency(100ms) 最多发 sendMaxCountOfEach(5) 条
            (WKOptions.m:48-49；WKMessageQueueManager.m:76,111)
          → 组 WKSendPacket：
              payload = AES(消息内容 encode)                   (coder/proto/WKSendPacket.m:42)
              msgKey = MD5(AES(签名串))                        (WKSendPacket.m:61-62)
          → WKConnectionManager sendPacket:
          → WKCoder encodeLM（lmproto 帧：1 字节固定头[包类型<<4|标志] + 变长剩余长度 + body）
                                                            (coder/WKCoder.m:25-42)
          → GCDAsyncSocket writeData                          (manager/WKConnectionManager.m:446-453)
  → 收到 SENDACK 后更新消息状态；失败走 WKRetryManager 重试
```

**连接与加密协商**：

```
connect → onlyConnect → loopGetAddrToConnect                  (WKConnectionManager.m:117/148/174-200)
  ├─ clusterOn=YES（WKAppConfig.m:34 默认开）：GET users/{uid}/im 取 tcp_addr (WKApp.m:332-346)
  └─ connectSocket:host:port（GCDAsyncSocket 明文 TCP，无 startTLS）          (WKConnectionManager.m:202-214)
didConnectToHost                                              (WKConnectionManager.m:348-351)
  → sendConnectPacket:uid:token:                              (WKConnectionManager.m:404-419)
      ├─ WKSecurityManager generateDHPair（Curve25519 生成密钥对） (manager/WKSecurityManager.m:39-41)
      └─ CONNECT 包字段：version/deviceFlag/deviceId/uid/token/clientTimestamp/clientKey(DH公钥)
          —— body 编码为明文拼接（writeVariableString），见 coder/proto/WKConnectPacket.m:42-60
收到 CONNACK → generateAesKey:salt:                           (WKConnectionManager.m:719 → WKSecurityManager.m:47-61)
  ├─ sharedKey = Curve25519(己方私钥, 服务端公钥)
  ├─ aesKey = MD5(sharedKey)[0..16]   （16 字节）
  └─ aesIV  = salt[0..16]
此后 SEND/RECV 包 payload 均 AES-128-CBC(PKCS7) 加密           (utils/WKAESUtil.m:60-79；WKSecurityManager.m:77-83)
```

**接收路径**：

```
socket didReadData                                            (WKConnectionManager.m:389-398)
  → unpacket（tempBufferData 粘包处理，按剩余长度字段切帧）     (WKConnectionManager.m:532+)
  → WKCoder decodeLM                                          (coder/WKCoder.m:45-65)
  → RECV 包处理：
      msgKey 校验：MD5(AES(veritifyString)) 与服务端下发值比对  (coder/proto/WKRecvPacket.m:60-61)
      payload AES 解密                                         (WKRecvPacket.m:66)
  → 离线拉取完成前暂存 tempPackets，syncConversations 完成后统一处理 (WKConnectionManager.m:220-280)
  → WKChatManager 解析 content、落库（FMDB+SQLCipher，WKMessageDB，1170 行）
  → 回 RECVACK；触发 onMessage 代理
  → UI：WKMessageListView 按 messageRegitry 注册的 content→cell 映射渲染 (WKApp.m:214-232)
```

**落库与迁移**：每个 uid 一个库文件 `Documents/wukongimdb/wukongim_{uid}.db`（`WKDB.m:73`；目录与前缀见 `WKOptions.m:9,37-38`），SQLCipher 打开时 `setKeyWithData`（`WKDB.m:88-91`，密钥问题见 §7.2）。表结构经 `WKDBMigrationManager` 跑 `WuKongIMSDK/Assets/Migrations/*.sql` 迁移（`WKDB.m:107-116`，迁移文件从 `201911270513.sql` 到 `202104081038.sql` 共 10+ 个，时间戳命名）。WuKongBase 侧另有 `wukongkit_` 前缀库（`WKKitDB.m:19`）。

**同步**：`WKSyncService sync`（`WuKongBase/Classes/Services/Service/WKSyncService.m:35-88`）通过 `invokes:WKPOINT_CATEGORY_SYNC` 收集所有注册为"可同步服务"的模块（联系人、会话、机器人、敏感词等），逐个 `needSync` 判断后并发执行并显示 HUD 进度；会话与离线消息同步由 SDK 内 `syncConversations:`（`WKConnectionManager.m:220-266`）走 REST `conversation/sync` 完成。

**WuKongBase 侧数据装配**：WuKongDataSource 模块提供 `WKMessageManagerDelegateImp`、`WKChannelDataManagerDelegateImp`、`WKGroupManagerDelegateImp`、`WKFileUploadTask`/`WKFileDownloadTask` 等实现（`Modules/WuKongDataSource/WuKongDataSource/Classes/`），把 SDK/WuKongBase 的 provider 回调对接到 REST 与文件服务——这是"SDK 不管业务、业务不管协议"的分层边界。

### 4.d APNs 推送与富通知

**device token 上报**：

```
didRegisterForRemoteNotificationsWithDeviceToken            (AppDelegate.m:132-158)
  → 转 hex 字符串（iOS13+ 读 bytes；以下用 description 去尖括号空格，AppDelegate.m:136-149）
  → NSLog 打印 token                                          (AppDelegate.m:151，日志泄露，见 §7.5)
  → 存 WKLoginInfo.deviceToken 并 save                        (AppDelegate.m:152-153)
  → POST user/device_token {device_token, device_type:"IOS", bundle_id}   (AppDelegate.m:155)
```

- 退到后台时上报角标数：`POST user/device_badge {badge: 全量未读}`（`WKApp.m:627-637`）。
- 登出时 `DELETE user/device_token`（`WKApp.m:608`）。
- 收到远程通知（静默/前台）路径：`AppDelegate.m:160-163` → `[WKApp.shared application:didReceiveRemoteNotification:...]`（`WKApp.m:267-272`）→ `WKSwiftModuleManager moduleDidReceiveRemoteNotification:`（`WKSwiftModuleManager.swift:112`）分发给各模块；ObjC 模块侧对应 `WKModuleManager.m:160-165`。

**通知扩展现状（与背景描述不符，需注意）**：

- `NotificationService/NotificationService.m:19-28` 是 **Xcode 模板级 stub**：仅在标题后追加 `" [modified]"` 字符串（`:25`），**没有任何消息解密/改写逻辑**，也不链接 WuKongIM SDK。这意味着服务端推送的 payload 本身已是可展示文本（或至少是可直出的密文提示），客户端不做通知侧端到端解密——与 Signal/WhatsApp 类"NSE 内解密改写"的方案不同。若产品宣称端到端加密，通知内容保护依赖服务端不落明文推送，**实际策略待验证（服务端代码不在本仓库）**。
- `NotificationContent/NotificationViewController.m` 同样为模板代码，`didReceiveNotification:` 仅把 body 赋给一个 label。
- entitlements 中 `aps-environment = development`（`TangSengDaoDao/TangSengDaoDao.entitlements`），上架前必须改为 production，当前仓库状态直接 Archive 会得到开发推送环境。
- 关联域名 `applinks:githubim.com`（同文件）配合 `WKApp appContinueUserActivity:`（`AppDelegate.m:174-177` → `WKApp.m:262-265` → `WKSwiftModuleManager didContinue:`）处理 Universal Links；自定义 scheme `botgate`（Info.plist `CFBundleURLTypes`）走 `appOpenURL:`（`AppDelegate.m:169-172` → `WKApp.m:258-260`）。

---

## 5. 对外接口

### 5.1 REST 端点清单（WKAPIClient，基址 `https://api.botgate.cn/v1/`）

以下端点字面量 grep 自 `Modules/` 与 `TangSengDaoDao/` 全部 .m 源码（按业务分组；带 `{}` 的为 stringWithFormat 动态拼接）。认证方式：HTTP header `token` + `bundle_id`（§4.b）。

**认证与账号**

| 方法 | 路径 | 用途 | 调用点示例 |
|---|---|---|---|
| POST | user/login | 登录（明文密码 JSON） | `WKLoginVM.m:49` |
| POST | user/register | 注册 | WuKongLogin |
| POST | user/pwdforget | 忘记密码重置 | WuKongLogin |
| POST | user/sms/registercode | 注册短信验证码 | WuKongLogin |
| POST | user/sms/forgetpwd | 找回密码短信 | WuKongLogin |
| POST | user/sms/login_check_phone / user/login/check_phone | 手机号校验登录 | WuKongLogin |
| GET | user/thirdlogin/authcode | 第三方授权登录取码 | WuKongLogin |
| GET | user/thirdlogin/authstatus | 第三方授权状态轮询 | WuKongLogin |
| PUT | user/current | 更新当前用户资料 | WuKongBase |
| PUT | user/my/setting | 更新用户设置 | WuKongBase |
| POST | user/chatpwd | 设置聊天密码 | WuKongBase |
| POST / DELETE | user/lockscreenpwd | 设置/删除锁屏密码 | WuKongBase |
| PUT | user/lock_after_minute | 锁屏时长 | WuKongBase |
| GET | user/qrcode | 我的二维码 | WuKongBase |
| GET | user/search | 搜索用户 | WuKongContacts |
| GET / POST | user/maillist | 通讯录匹配（上传手机联系人） | WuKongContacts |
| GET / POST | user/online | 在线状态查询/上报 | WuKongBase |
| POST | user/pc/quit | 踢 PC/web 端下线 | WuKongBase |

**设备与推送**

| 方法 | 路径 | 用途 | 调用点 |
|---|---|---|---|
| POST | user/device_token | APNs token 上报 | `AppDelegate.m:155` |
| DELETE | user/device_token | 登出注销 | `WKApp.m:608` |
| POST | user/device_badge | 角标数上报 | `WKApp.m:632` |
| GET | user/devices/{uuid} | 查询设备数字 id | `WKApp.m:445` |

**会话与消息（同步族）**

| 方法 | 路径 | 用途 |
|---|---|---|
| POST | conversation/sync | 最近会话同步（版本号+syncKey 增量） |
| POST | conversation/syncack | 会话同步回执 |
| POST | conversation/extra/sync | 会话扩展同步 |
| POST | message/channel/sync | 频道历史消息同步 |
| POST | message/extra/sync | 消息扩展（编辑等）同步 |
| POST | message/offset | 消息偏移上报 |
| POST | message/reminder/sync | 提醒项同步（@我/草稿） |
| POST | message/reminder/done | 提醒项完成 |
| POST | message/typing | 正在输入状态 |
| PUT | message/voicereaded | 语音已读 |
| PUT | coversation/clearUnread | 清未读（注意服务端路由拼写为 "coversation"） |
| DELETE | message | 删除消息 |
| GET | message/prohibitwords/sync | 违禁词同步 |
| GET | message/sync/sensitivewords | 安全提醒敏感词同步 |

**群组 / 好友 / 机器人 / 表情 / 其他**

| 方法 | 路径 | 用途 |
|---|---|---|
| GET | group/my | 我的群列表 |
| POST | group/create | 建群 |
| POST | friend/apply / friend/sure | 好友申请/确认 |
| PUT | friend/remark | 好友备注 |
| POST | robot/inline_query | 机器人 inline 查询 |
| POST | robot/sync | 机器人同步 |
| POST | search/global | 全局搜索 |
| GET / POST / DELETE | sticker/user、sticker/user/sticker | 收藏表情增删查 |
| PUT | sticker/user/front | 表情置顶 |
| GET | file/upload?type=sticker | 表情上传取地址 |
| GET | invite、PUT invite/status | 邀请码 |
| GET | common/appconfig | 远程配置（`WKApp.m:150-158` WKAppRemoteConfig） |
| GET | common/appmodule | 远程模块开关 |
| GET | common/countries | 国家区号 |
| GET | common/appversion/iOS/{version} | 版本检查（`AppDelegate.m:109`） |
| GET | openapi/authcode | 开放平台授权码（JS 桥 auth，`WKWebViewService.m:106`） |
| GET | apps/{appID} | 应用信息（授权弹窗，`WKWebViewService.m:82`） |
| GET | users/{uid}/im | 集群模式取 IM tcp_addr（`WKApp.m:334`） |

**文件传输**：两段式——先 `GET {fileBaseUrl}file/upload?path={path}&type=chat` 取上传 URL（`WKAPIClient.m:223-225`），再 multipart POST 上传（`WKAPIClient.m:154-209, 243-284`）；下载走 `createDownloadTask:storePath:`（`WKAPIClient.m:227-241`）。

### 5.2 长连接协议配置（WuKongIMSDK）

- **帧格式**：自研二进制协议（代码内称 lmproto）：1 字节固定头（高 4 位包类型 + syncOnce(2)/showUnread(1)/noPersist(0) 标志位）+ 变长剩余长度（7bit 编码）+ body（`WKCoder.m:25-42` 编码、`:45-65` 解码）。
- **包类型族**：CONNECT / CONNACK / SEND / SENDACK / RECV / RECVACK / PING / PONG / DISCONNECT（`coder/proto/` 目录各一个类）。
- **默认地址**：`127.0.0.1:6666`（`WKOptions.m:15-16`）；生产环境由 `users/{uid}/im` 动态下发 `tcp_addr`（集群模式，`WKApp.m:332-346`）。
- **心跳**：60s（`WKOptions.m:18`），PING 单字节（`WKCoder.m:32-34`）。
- **离线消息**：上限 300 条（`WKOptions.m:33`）；连接后先 `syncConversations` 再处理缓存包（`WKConnectionManager.m:220-280`）。
- **重连**：指数退避 base=2s、factor=2、cap=60s（`WKConnectionManager.m:104-109` 的 WKBackoffBuilder）。
- **传输加密**：Curve25519 DH 协商 → AES-128-CBC(PKCS7) 加密 payload；msgKey = MD5(AES（签名串）) 做完整性校验（`WKSecurityManager.m`、`WKSendPacket.m:42,61-62`、`WKRecvPacket.m:60-66`）。**TCP 通道本身无 TLS；CONNECT 包（含 token）body 为明文拼接**（`WKConnectPacket.m:42-60`），详见 §7.1。
- **端对端**：UI 层注册了"端对端加密提示"（`WKEndToEndEncryptHitContent`）与"解密失败"（`WKSignalErrorContent`，文案"[消息无法解密，因为双方密钥有发送变更]"，`WKAppConfig.m:66`）两种消息类型，表明产品层面有 Signal 风格 E2E 功能设计；登录响应还下发 `rsa_public_key`（`WKLoginVM.m:31`）。E2E 的完整实现链路（密钥交换/会话建立）未在本次走读范围内完全展开，**待验证**。

---

## 6. 部署与运维

- **Podfile 结构**：见 §2.1。`use_frameworks!`；5 个本地 path pod + 4 个 git fork pod + 官方源 pod。
- **签名痕迹**：post_install 硬编码 Team ID `H8PU463W68`（`Podfile:6`）；entitlements 仍是 `aps-environment development` 与 `applinks:githubim.com`；URL scheme 注册为 `botgate`（Info.plist `CFBundleURLTypes`，其中 `CFBundleURLName` 残留 "webchat" 字样，疑似早期从微信接入模板拷贝）。
- **CI 痕迹**：仓库内未发现 `.github/workflows`、fastlane、xcodebuild 脚本等 CI 配置（待验证是否依赖外部 CI/手工打包）。
- **Localization**：主 app 的 `zh-Hans.lproj/Localizable.strings` 与 `en.lproj/Localizable.strings` 几乎为空（1 行/5 行）。实际文案在 WuKongBase 的 `Assets/Lang/`：**`zh-Hans.lproj/Localizable.strings` 是 1 字节空文件**，`en.lproj` 578 行、以中文为 key 映射英文（如 `"通用"="General";`）。机制：`LLang(a)` 宏（`WuKongBase.h:156`）→ `NSString+WKLocalized.m:22-49`，按 `WKApp.config.langue` 从模块 bundle 的 `Lang/{lang}.lproj` 取 `NSLocalizedStringFromTableInBundle`，取不到回退中文字符串本身——即**中文是硬编码在源码里的默认语言**，英文靠翻译表覆盖。语言切换持久化在 loginInfo.extra（`WKAppConfig.m` setStyle 等处）。
- **Bugly 接入**：`WKApp.m:234-248` `traceConfig`：`[Bugly startWithAppId:@"82f8dd98ff" config:]`（`:244`），登录后 `setUserIdentifier:uid`（`:246`）；DEBUG 下 `reportLogLevel = BuglyLogLevelDebug`、Release 下 Warn（`:236-242`）。**AppID 硬编码明文**。
- **日志体系**：CocoaLumberjack 配置 OSLogger（iOS10+）/TTY + `DDFileLogger`（24h 滚动、最多 4 个文件，`WKLogs.m:16-35`），`ddLogLevel = DDLogLevelAll`（`WKLogs.h:12`）。注意 `WKLogs.m:25` 的 `if (logsDirectory)` 条件疑似写反：传入 nil（`WKApp.m:302` 实际调用方式）时不进入分支、logsDirectory 保持 nil，DDFileLogger 落到默认目录——功能上仍工作，但自定义目录逻辑是死的（小 bug）。
- **业务日志宏**：`WKLogInfo/WKLogDebug/WKLogError/WKLogWarn` 在 DEBUG 与 Release 两个条件分支里**都是 NSLog**（`WKLogs.h:17-30`），等于完全没有日志开关，见 §7.5。
- **崩溃/性能调试**：DEBUG 下状态栏 FPS 显示（`WKApp.m:250-256`，FPSCounter）。
- **更新检查**：`common/appversion/iOS/{version}` + `SELUpdateAlert` 强制/非强制更新弹窗（`AppDelegate.m:106-126`；30 分钟节流存 NSUserDefaults `lastCheckUpdateTime`）。
- **远程配置**：`WKAppRemoteConfig`（`WKApp.m:150-158`）拉 `common/appconfig` 与 `common/appmodule`，可做模块级开关，是线上运营的灰度手段。
- **版本迁移**：DB 结构靠时间戳 SQL 迁移文件（`WuKongIMSDK/Assets/Migrations/`），`WKDBMigrationManager` 记录已执行版本（`WKDB.m:95-116`）。

---

## 7. 安全审计

标注约定：**[CONFIRMED]** = 静态代码可直接证实；**[PLAUSIBLE]** = 代码证据支持但需运行时/服务端配合才能坐实影响。仅静态审查，未做动态验证。

### 7.1 传输安全

**[CONFIRMED]（高危）REST 层完全关闭证书校验**

`Modules/WuKongBase/WuKongBase/Classes/Services/Base/WKAPIClient.m:43-48`：

```objc
if([config.baseUrl hasPrefix:@"https"]) {
    AFSecurityPolicy *securityPolicy = [AFSecurityPolicy policyWithPinningMode:AFSSLPinningModeNone];
    securityPolicy.allowInvalidCertificates = YES;
    securityPolicy.validatesDomainName = NO;
    _sessionManager.securityPolicy = securityPolicy;
}
```

只要 baseUrl 是 https（当前硬编码就是 https，`AppDelegate.m:20`），就接受任意证书、不校验域名。登录接口 `user/login` 明文传输密码 JSON（`WKLoginVM.m:49`）、所有请求头携带 token（`WKApp.m:532`）——**在恶意 Wi-Fi/恶意代理场景下，攻击者可窃取账号密码与 token，进而完全接管账号**。这是全仓最高危发现。

**[CONFIRMED]（高危）内嵌 WebView 同样盲目信任任意证书**

`WKWebViewVC.m:392-404`：`didReceiveAuthenticationChallenge` 对 `NSURLAuthenticationMethodServerTrust` 直接构造 `NSURLCredential initWithTrust:` 并 `NSURLSessionAuthChallengeUseCredential`，不做任何评估。WebView 内打开的所有 https 页面均可被中间人。结合 §7.4 的 JS 桥，危害放大。

**[CONFIRMED]（中危）IM 长连接 TCP 无 TLS，且 CONNECT 包明文携带 token**

- `WKConnectionManager.m:202-214` 使用 GCDAsyncSocket 明文 TCP 直连，全文件无 `startTLS` 调用。
- 连接建立后立即 `sendConnectPacket:uid:token:`（`WKConnectionManager.m:348-351`），而 AES 密钥要等 CONNACK 回来才生成（`WKConnectionManager.m:719`）。
- CONNECT body 编码为**明文变长字符串拼接**：uid、token、deviceId、DH 公钥依次 `writeVariableString`（`coder/proto/WKConnectPacket.m:42-60`），无加密。
- 结论：**每次 IM 连接/重连，imToken 都在 TCP 上明文传输一次**；任何能监听该链路的攻击者可重放 token 接管 IM 会话（服务端若有设备绑定可缓解，待验证）。好消息是 SEND/RECV 的业务 payload 有 DH+AES 加密（`WKSecurityManager.m`），消息内容本身不明文；但 DH 无身份认证（无证书/无指纹校验），理论上可被主动中间人双向代理（[PLAUSIBLE]，利用难度取决于服务端校验）。

**[CONFIRMED]（中危）ATS 全局豁免 + URL 强补 http**

主 app `Info.plist`：`NSAppTransportSecurity/NSAllowsArbitraryLoads = true`，允许任意明文 http。`WKWebViewVC.m:57-59` 还会把不带 scheme 的 URL 强行补 `http://`（明文）。

### 7.2 数据存储安全

**[CONFIRMED]（高危）SQLCipher 密钥 = 用户 uid，强度形同虚设**

`Modules/WuKongIMiOSSDK/WuKongIMSDK/Classes/db/WKDB.m:78-91`：

```objc
NSString *dbKey = uid;
...
FMDatabaseQueue *fmdb = [FMDatabaseQueue databaseQueueWithPath:dbPath];
self.dbQueue = [WKFMDatabaseQueue databaseQueue:fmdb] ;
[self.dbQueue inDatabase:^(FMDatabase * _Nonnull db) {
    [db setKeyWithData:[dbKey dataUsingEncoding:NSUTF8StringEncoding]];
}];
```

数据库密钥直接是用户 uid（uid 在登录响应中下发、UI 展示、分享名片/二维码也会携带），未经过任何 KDF 派生、未存 Keychain。任何能读到沙盒 db 文件的攻击者（越狱设备、物理取证、未加密备份），只要知道 uid（极易获得）即可解密全部聊天记录。**SQLCipher 在此仅提供"合规观感"，不提供实际机密性**。另注意 `WKDB.m:74-76` debug 模式还把数据库路径打到日志。

**[CONFIRMED]（高危）token/敏感信息存 NSUserDefaults 而非 Keychain**

`WKLoginInfo.m:131-135` 将 uid、token、imToken、deviceToken 及 extra（含手机号、`chat_pwd`、`lock_screen_pwd`、`rsa_public_key` 等，见 `WKLoginVM.m:61-84`）NSKeyedArchiver 后写入 NSUserDefaults。NSUserDefaults 的保护级别远低于 Keychain，未加密备份可读出；设备被越狱后直接明文可读。项目里其实已有 Keychain 封装（`WKKeychainUtil`，被 `UIDevice+Utils.m:10,20-26` 用于 IDFV），迁移成本很低。

**[PLAUSIBLE]（中危）锁屏密码明文存储**

锁屏密码以 `lock_screen_pwd` 键明文存于 loginInfo.extra（`WKLoginVM.m:69-71`），随 NSUserDefaults 落盘；聊天密码 `chat_pwd` 同理（`WKLoginVM.m:68`）。本地校验逻辑未见哈希（待验证校验是本地比对还是走服务端）。若服务端也是明文/可逆存储，风险放大——服务端行为待验证。

### 7.3 硬编码与配置

**[CONFIRMED] 硬编码生产服务器与 HTTPS 开关**

`AppDelegate.m:19-20`：`SERVER_IP @"api.botgate.cn"`、`HTTPS_ON true`。换环境必须改源码重新编译；无 xcconfig/构建变体区分。`HTTPS_ON` 若为 false 则全站明文 http（配合 ATS 豁免可直接工作）。

**[CONFIRMED] 硬编码 Bugly AppID、开发者 Team ID**

Bugly AppID `82f8dd98ff`（`WKApp.m:244`）；Team ID `H8PU463W68`（`Podfile:6`）。

**[CONFIRMED] entitlements 残留 development 推送环境**

`TangSengDaoDao/TangSengDaoDao.entitlements`：`aps-environment = development`。若以此打包分发，生产推送不可用；也反映发布流程依赖手工改配置，容易出错。

### 7.4 WebView / JS 桥（呼应 TangSengDaoDaoJSBridge 仓库）

**[CONFIRMED]（中危）JS 桥对任意页面开放原生能力，无来源校验**

`WKWebViewVC.m:243-247` 对加载的**每个页面**都挂 vendored WebViewJavascriptBridge 并注册 handler（`WKWebViewService.m:15-101`）：

| handler | 能力 | 位置 |
|---|---|---|
| `quit` | 关闭当前页面 | `WKWebViewService.m:28-30` |
| `getChannel` | 获取当前所在频道 id/type | `:33-38` |
| `chooseConversation` | 拉起原生会话选择器并把结果回传 JS | `:41-55` |
| `showConversation` | 按 channel_id/type 直接打开任意会话 UI（可 replace 当前页） | `:58-70` |
| `auth` | 拉取 app 信息→弹授权框→把 `openapi/authcode` 交给 JS | `:72-114` |

没有 URL 白名单/origin 检查：**任何被打开的第三方网页（含聊天中点开的链接）都能调用这些 handler**。`auth` 虽有用户确认弹窗（`WKUserAuthView`），但 `showConversation`/`getChannel`/`chooseConversation` 无任何确认。结合 §7.1 WebView 证书盲从，恶意页面可被注入后静默调用桥接能力（影响升级路径 [PLAUSIBLE]）。

**[CONFIRMED]（中危）任意非 http(s) scheme 直接 openURL**

`WKWebViewVC.m:361-369`：页面内跳转到非 http(s) URL 时直接 `[[UIApplication sharedApplication] openURL:]`，无 scheme 白名单。恶意网页可尝试调起任意已注册 scheme（含本 app 的 `botgate://`，以及电话/短信等）。

**[PLAUSIBLE] decidePolicy 中打印完整请求头**

`WKWebViewVC.m:349` `NSLog(@"%@", navigationAction.request.allHTTPHeaderFields)`——若请求带 Cookie/凭证会进日志。

### 7.5 日志泄露

**[CONFIRMED]（中危）Release 构建保留全部 NSLog，网络层全量打印请求/响应**

`WKLogs.h:17-30`：`WKLogInfo/WKLogDebug/WKLogError/WKLogWarn` 在 `#ifndef __OPTIMIZE__` 与 `#else` 两个分支里**都是 NSLog**（条件编译形同虚设）。于是：

- `WKAPIClient.m:404-417`：每个请求打印完整 URL 与参数（`logRequestStart`），响应体全量打印（`logRequestEnd`）——**登录密码、消息内容、手机号、token 相关响应全部进系统日志**；
- `AppDelegate.m:151`：打印 APNs device token；
- `WKConnectionManager.m:391-397`：debug 分支打印原始包数据（`isDebug` 默认 true，`WKOptions.m:17`）；
- `WKWebViewVC.m:349,438,454`：请求头、滚动偏移等。

iOS 上 NSLog/os_log 可被同设备授权工具（Console、idevicesyslog）读取；企业分发/越狱场景风险更高。

### 7.6 其他攻击面

**[CONFIRMED] Bugly 数据外发**

Bugly（腾讯）崩溃 SDK 以 Debug 级日志上报（DEBUG 包，`WKApp.m:236-241`），并绑定用户 uid（`WKApp.m:246`）。崩溃上下文可能夹带上述敏感日志；数据流向腾讯服务器，隐私政策需披露（隐私协议 URL 指向自家服务器 `AppDelegate.m:68`，其内容是否披露 Bugly 待验证）。

**[CONFIRMED] Aspects 为死依赖**

Podfile 引入 Aspects（AOP），但自有代码（`Modules/`、`TangSengDaoDao/`）无任何 `aspect_hookSelector` 调用（全仓 grep 仅命中 Pods 内 Aspects 自身）。当前无实际 hook 面，但保留该库意味着未来一行代码即可全局劫持任意方法，从供应链与 App Store 审核角度建议移除。

**[PLAUSIBLE] SQL 拼接注入面**

`WKDB.m` 的 `deleteTable:whereFormat:`（`:317-328`）、`QueryTable:whereFormat:`（`:389-412`）、`updateTable:...whereFormat:`（`:353-387`）等以 `stringWithFormat` 拼 WHERE 子句。是否可被远端数据（频道名、搜索词、消息字段）注入取决于各调用点传参——抽查显示调用点多传常量格式串，**未证实存在可利用注入**，但该模式本身危险，建议全部参数化。

**[PLAUSIBLE] 弱 KDF 与弱完整性算法**

AES 会话密钥派生用 `MD5(sharedKey)[0..16]`（`WKSecurityManager.m:53-54`），msgKey 也用 MD5（`WKSendPacket.m:62`、`WKRecvPacket.m:61`）。MD5 作 KDF/MAC 已不合规（长度截断+无 key 的 MD5 不是 MAC）；建议 HKDF-SHA256 + HMAC。实际可攻击性待验证（需结合服务端实现）。

**[CONFIRMED] 设备标识双轨**

`UIDevice+Utils.m:16-33`（IDFV → Keychain，做法合理）；`WKConnectionManager.m:421-438`（`deviceUUIDv2:{uid}` 存 NSUserDefaults，重装/清数据即变，作为设备标识可靠性一般）——属设计一致性问题而非漏洞。

**[CONFIRMED] 无代码混淆/反调试/越狱检测痕迹**

全仓未见反调试、完整性校验、越狱检测（静态审查范围内未发现，待验证是否有编译期注入）。对 IM 类应用属可接受的基线缺失，但与"安全通讯"定位不匹配。

### 7.7 安全发现汇总表

| # | 级别 | 状态 | 发现 | 位置 |
|---|---|---|---|---|
| 1 | 高 | CONFIRMED | REST 关闭证书校验与域名校验 | `WKAPIClient.m:43-48` |
| 2 | 高 | CONFIRMED | WebView 无条件信任任意证书 | `WKWebViewVC.m:392-404` |
| 3 | 高 | CONFIRMED | SQLCipher 密钥=uid 明文、无 KDF | `WKDB.m:78,89` |
| 4 | 高 | CONFIRMED | token/密码等敏感数据存 NSUserDefaults | `WKLoginInfo.m:123-135` |
| 5 | 高 | CONFIRMED | 登录密码明文 JSON + #1 组合可窃密 | `WKLoginVM.m:49` |
| 6 | 中 | CONFIRMED | IM CONNECT 包明文携带 token，TCP 无 TLS | `WKConnectPacket.m:42-60`、`WKConnectionManager.m:202-214,348-351` |
| 7 | 中 | CONFIRMED | Release 全量 NSLog 请求/响应/token | `WKLogs.h:17-30`、`WKAPIClient.m:404-417`、`AppDelegate.m:151` |
| 8 | 中 | CONFIRMED | JS 桥无 origin 校验开放原生能力 | `WKWebViewService.m:15-101`、`WKWebViewVC.m:243-247` |
| 9 | 中 | CONFIRMED | NSAllowsArbitraryLoads = true | `TangSengDaoDao/Info.plist` |
| 10 | 中 | CONFIRMED | 任意非 http(s) scheme 直接 openURL | `WKWebViewVC.m:361-369` |
| 11 | 中 | CONFIRMED | 锁屏密码/聊天密码明文随 loginInfo 落盘 | `WKLoginVM.m:68-71` |
| 12 | 低 | CONFIRMED | 硬编码服务器/Bugly AppID/Team ID/dev 推送环境 | `AppDelegate.m:19-20`、`WKApp.m:244`、`Podfile:6`、entitlements |
| 13 | 低 | CONFIRMED | MD5 作 KDF / msgKey | `WKSecurityManager.m:53-54,64-75`、`WKSendPacket.m:62` |
| 14 | 低 | CONFIRMED | Bugly 外发绑定 uid | `WKApp.m:234-248` |
| 15 | 低 | CONFIRMED | DH 密钥协商无身份认证（MITM 理论面） | `WKSecurityManager.m:39-61` |
| 16 | 待查 | PLAUSIBLE | SQL whereFormat 拼接注入面 | `WKDB.m:317-412` |
| 17 | 待查 | PLAUSIBLE | URL 编码用废弃 API 致参数错误 | `WKAPIClient.m:142-144` |

---

## 8. 代码质量评估

### 8.1 TOP 文件（行数，不含 Pods/Example/Tests）

| 行数 | 文件 | 说明 |
|---|---|---|
| 4729 | `WuKongBase/.../TelegramUtils/Display/Source/ListView.swift` | Telegram 移植（vendor） |
| 2448 | `.../TelegramUtils/ContextUI/Sources/ContextController.swift` | vendor |
| 2133 | `.../TelegramUtils/Display/Source/TextNode.swift` | vendor |
| 1963 | `.../TelegramUtils/Display/Source/ContainedViewLayoutTransition.swift` | vendor |
| 1824 | `WuKongIMiOSSDK/.../manager/WKChatManager.m` | **自有代码最大文件**：消息收发/同步/编辑/删除/转发全在一个类 |
| 1782 | `WuKongBase/.../WKApp.m` | 上帝对象：配置、导航、推送、Bugly、屏幕保护、消息注册、前后台…… |
| 1633 | `.../TelegramUtils/Display/Source/Navigation/NavigationController.swift` | vendor |
| 1599 | `WuKongBase/.../Sections/Conversation/WKMessageListView.m` | 消息列表 UI |
| 1489 | `.../TelegramUtils/Display/Source/NavigationBar.swift` | vendor |
| 1450 | `.../TelegramUtils/Display/Source/GridNode.swift` | vendor |
| 1300 | `.../TelegramUtils/Display/Source/WindowContent.swift` | vendor |
| 1269 | `.../TelegramUtils/ReactionSelectionNode/.../ReactionContextNode.swift` | vendor |
| 1170 | `WuKongIMiOSSDK/.../db/WKMessageDB.m` | 消息表全部 SQL |
| 1164 | `WuKongBase/.../Sections/Messages/WKMessageCell.m` | cell 基类 |
| 1160 | `WuKongBase/.../Services/Category/NSAttributedString+Markdown.m` | |
| 1050 | `WuKongBase/.../Sections/Conversation/Input/WKConversationInputPanel.m` | 输入面板 |

自有代码中 `WKChatManager.m`（1824 行）与 `WKApp.m`（1782 行）两个巨型类集中了过多职责，是最需要拆分的位置；TelegramUtils 占了 TOP15 中的 9 席，vendor 代码体量已反超业务核心代码。

### 8.2 ObjC/Swift 混编趋势

- 框架与业务主体是 2019 年起步的 ObjC（`Created by tt on 2019/... Copyright © 2019 xinbida` 头注释遍布），Swift 主要沿两条线进入：
  1. 后期新增基础设施（`WKSwiftModuleManager.swift`、少量 UI/服务）；
  2. **整块 vendor Telegram iOS 的 Swift 库**（约 200 个 Swift 文件）。
- 桥接方式：ObjC→Swift 经自动生成的 `WuKongBase-Swift.h`（`WKApp.m:80` import）；Swift→ObjC 经（近乎为空的）`TangSengDaoDao-Bridging-Header.h` 与各 pod 内部 header。
- 趋势判断：作者主力语言仍是 ObjC，Swift 增量有限且多为移植代码；随着 Swift 功能增加，`@objc` 暴露面与模块耦合成本会上升。

### 8.3 Pods 入库的利弊

- **利**：克隆即可构建（对依赖 4 个个人 fork git pod 的项目尤为重要，规避 fork 仓库失效风险）；无 CI 也能稳定出包。
- **弊**：189MB 仓库体积；依赖升级 diff 淹没业务变更；`Pods/` 内代码可能被误改且难以审查；Pods 实际内容与 Podfile.lock 的一致性无保障（未逐项核对，待验证）。

### 8.4 测试

- 主 app 无 XCTest target（pbxproj 中仅 3 个 target：主 app + 两个通知扩展）。
- 各本地 pod 的 `Tests/` 为 stub：WuKongBase/Tests 下仅 `AvatarLayoutTests.m` + 一个手工 shell 脚本（`run-avatar-layout-tests.sh`），非标准 XCTest 体系；`Example/` 工程里的 Tests 为 CocoaPods 模板。
- 关键逻辑（协议编解码 `WKCoder`、加解密 `WKSecurityManager`、DB 迁移 `WKDBMigrationManager`、消息队列 `WKMessageQueueManager`）**零测试覆盖**，协议升级与重构全靠人工回归。

### 8.5 注释与文档

- 中文注释覆盖率尚可，关键流程（连接、重连、同步、面板扩展）都有注释；但存在大量注释掉的死代码，如 `WKChatManager.m:292-316` 整段旧发送实现、`WKDB.m:81-82` 旧加密方案（WKFMEncryptHelper）、`WKLoginInfo.m:29-44` 旧单例、`WKApp.m:89-91` 旧 PINRemoteImage 引用、`WKLogs.m:18` ASL 注释。
- `WuKongBase.podspec` 的 summary/description 仍是模板占位（`WuKongBase.podspec:13-23`），README 仅一行外链；模块级文档缺失。
- 复制粘贴/历史痕迹：`WKLogs.m:3-6` 头注释还是老项目名 "TalkClient3 / aiti"；Info.plist `CFBundleURLName` 为 "webchat"；服务端路由拼写错误 "coversation" 已被客户端沿用成为线上契约；邀请文案指向 `www.githubim.cn`（`WKAppConfig.m:89`），与 applinks 域 `githubim.com`、生产 API 域 `api.botgate.cn` 三个域并存，品牌演进痕迹明显（githubim → botgate → tsdaodao）。

### 8.6 其他质量问题

- `WKLoginInfo shared` 非线程安全：`WKLoginInfo.m:45-49` 无锁判空赋值，`load` 直接替换 `_instance`（`WKLoginInfo.m:123-128`），多线程下可能拿到不同实例。
- `stringByAddingPercentEscapesUsingEncoding` 已废弃多年仍用于 URL 编码（`WKAPIClient.m:142-144`），对 `+`、`&`、`=` 等字符编码不正确，可能引发间歇性参数错误 [PLAUSIBLE bug]。
- `WKLogs.m:25` `if (logsDirectory)` 条件疑似写反（见 §6）。
- `WKAPIClient.m:72-113` 的 `taskGET:...callback:` 在错误分支回调 `callback(error,nil)` 而非加工后的 `er`（`:109`），errorHandler 的加工结果被丢弃——与 promise 版行为不一致 [PLAUSIBLE bug]。
- 做得好的部分：内存警告时清图片缓存（`WKApp.m:193-197`）、断网屏幕保护/锁屏保护（`WKApp.m:426-430`）、截屏通知消息（WKScreenshotContent）、录屏状态监听（`WKApp.m:172-175`）、消息重试/限速队列、DB 版本迁移机制，隐私与健壮性考虑在功能层是到位的。

---

## 9. 风险与改进建议

| 类别 | 风险 | 严重度 | 建议 |
|---|---|---|---|
| 安全 | REST/WebView 证书校验全关（§7.1 #1#2） | 严重 | 立即移除 `allowInvalidCertificates=YES`/`validatesDomainName=NO`，删除 WebView 的 challenge 盲从实现；如确有自签调试需求，用构建宏限定 DEBUG；评估加证书 pinning |
| 安全 | SQLCipher 密钥=uid（§7.2 #3） | 高 | 改为随机 256bit 密钥 + Keychain 存储 + 与 uid 绑定；旧库做一次性迁移重加密 |
| 安全 | token 存 NSUserDefaults（§7.2 #4） | 高 | 迁移 Keychain（`kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`，项目已有 `WKKeychainUtil` 可复用），首启迁移旧数据并删除 |
| 安全 | IM CONNECT 明文 token、TCP 无 TLS（§7.1 #6） | 高 | 优先在 socket 层加 TLS（需服务端支持）；过渡期至少对 CONNECT 包用预共享密钥/RSA（登录响应已下发 `rsa_public_key`）加密 |
| 安全 | Release 全量 NSLog（§7.5 #7） | 高 | WKLog 宏在 `__OPTIMIZE__` 下置空或接入 CocoaLumberjack 并分级；网络层日志对 token/密码/手机号脱敏 |
| 安全 | JS 桥无 origin 白名单、任意 scheme openURL（§7.4 #8#10） | 中 | 按 host 白名单注册 handler；openURL 增加 scheme 白名单；`auth` handler 增加调用来源展示 |
| 安全 | ATS 全局豁免 + URL 强补 http（§7.1 #9） | 中 | 移除 NSAllowsArbitraryLoads，按域配置例外；URL 补全改 https |
| 安全 | 锁屏/聊天密码明文落盘（§7.1 #11） | 中 | 本地仅存哈希或不存；校验逻辑移服务端 |
| 安全 | MD5 作 KDF/MAC、DH 无身份认证（§7.6 #13#15） | 中 | HKDF-SHA256 + HMAC；协商过程绑定服务端身份（证书指纹或服务端签名） |
| 合规 | entitlements dev 推送环境 / Bugly 数据外发披露（§6、§7.6） | 中 | 发布流水线固化 `aps-environment=production`（按配置切换）；隐私政策披露 Bugly 数据收集 |
| 依赖 | 4 个个人 fork pod + lottie 2.5.3 + SQLCipher 4.5.4 等陈旧组件（§2.3） | 中 | fork 转组织账号并打 tag；制定升级计划（lottie 3+/4+、SQLCipher 最新、webp 最新）；死依赖（Aspects/Socket.IO/SocketRocket）直接移除 |
| 架构 | WKApp/WKChatManager 上帝类、point 机制无编译期检查（§3.3、§8.1） | 中 | 按启动、推送、同步、消息域拆分；point 定义收敛为常量+协议文档；为 coder/security/db/queue 补单元测试 |
| 工程 | Pods 入库 189MB（§8.3） | 低 | 中期迁出 Pods（配合私有 specs 源或 vendor cache），先确保 fork 仓库已归档备份 |
| 工程 | 无 CI/无测试/环境靠改源码（§6、§7.3） | 中 | 引入 xcconfig 区分 dev/stage/prod + fastlane 或 GitHub Actions；至少覆盖协议编解码与加解密 |
| 兼容性 | iOS 12 部署目标（§2.4） | 低 | 评估提升到 iOS 13/14，清理 @available 分支与废弃 API（`stringByAddingPercentEscapesUsingEncoding`、`UIUserNotificationSettings` 旧路径） |
| 服务端契约 | "coversation" 拼写已固化（§8.5） | 低 | 服务端新增正确拼写路由，客户端双写过渡后下线旧路由 |
| 品牌一致性 | githubim.cn/githubim.com/botgate.cn/tsdaodao.com 多域并存（§8.5） | 低 | 统一关联域名与邀请链接，避免用户混淆与钓鱼面 |

---

## 10. 模块文件清单（附录 A）

### 10.1 主 app target（TangSengDaoDao/）

| 文件 | 行数 | 职责 |
|---|---|---|
| `main.m` | 16 | `UIApplicationMain` 入口（`main.m:12-15`） |
| `AppDelegate.m` | 180 | 启动装配、硬编码服务器、版本检查、deviceToken 上报、URL/UniversalLink 分发 |
| `WKMainTabController.m` | 73 | 三 tab 容器 + 触感反馈 |
| `ViewController.m` | 模板 | 未使用（Xcode 模板遗留） |

pbxproj Sources 段确认主 target 编译的 .m 即上述 4 个（`project.pbxproj:35-41`）。

### 10.2 WuKongLogin（登录模块）

| 文件 | 行数 | 职责 |
|---|---|---|
| `Classes/Login/WKLoginModule.m` | ~50 | `@WKModule` 注册；`login.show`、扫码授权登录 point（`:26-47`） |
| `Classes/Login/Src/WKLoginVC.m` | 101 | 手机号密码登录 VC；成功后 invoke `login.success`（`:56-61`） |
| `Classes/Login/Src/WKLoginVM.m` | 98 | `user/login` 请求 + `handleLoginData:` 登录数据落地（`:47-94`） |
| `Classes/Login/Src/WKLoginView.m` | — | 登录表单 UI |
| `Classes/Login/Src/WKThirdLoginVC.m` | 258 | 第三方授权登录（web 扫码轮询） |
| `Classes/Login/Src/WKRegisterVC/VM.m`、`WKRegisterNextVC.m` | — | 注册 |
| `Classes/Login/Src/WKForgetPasswordVC/VM.m` | — | 找回密码 |
| `Classes/Login/Src/WKLoginPhoneCheckVC.m`（141）、`WKLoginPhoneCheckStartVC.m`（98） | — | 短信验证码登录 |
| `Classes/Login/Src/WKGrantLoginVC/VM.m` | — | 扫码授权登录确认（loginConfirm） |
| `Classes/Login/Src/WKAuthWebViewVC.m` | — | 授权 WebView（未单独审计，待验证） |
| `Classes/Login/Src/WKLoginSettingVC.m` | 252 | 登录页设置（服务器/语言） |
| `Classes/Login/Src/WKCountrySelectVC.m` | — | 国家区号选择（`common/countries`） |

### 10.3 WuKongContacts（联系人模块）

`Classes/Src/` 下：`WKContactsVC`（通讯录 tab）、`WKContactsAddVC`、`WKContactsCell`、`WKContactsFriendCell`、`WKContactsAddFunctionItemCell`、`WKContactsAddMyShortCell`（我的短号）等；`Classes/Service/` 提供好友申请/通讯录匹配服务（`friend/apply`、`friend/sure`、`user/maillist` 端点）。`WKContactsModule.m` 注册模块 point。

### 10.4 WuKongDataSource（数据源装配层）

| 文件 | 职责 |
|---|---|
| `WKDataSourceModule.m` | 模块入口 |
| `WKMessageManagerDelegateImp.m` | WuKongBase 消息管理器 delegate 实现（消息同步/搜索结果装配） |
| `WKChannelDataManagerDelegateImp.m` | 频道资料数据源 |
| `WKGroupManagerDelegateImp.m` | 群资料数据源 |
| `WKFileUploadTask.m` / `WKFileDownloadTask.m` | 聊天文件两段式上传/下载任务 |
| `WKDataSourceModel.m` | 装配用模型 |

### 10.5 WuKongIMiOSSDK（IM 协议栈）

| 子目录 | 内容 | 关键文件（行数） |
|---|---|---|
| `Classes/coder` | 帧编解码 + body coder 管理 | `WKCoder.m`（110）、`WKPakcetBodyCoderManager.m`、`WKData.m`（变长编码读写） |
| `Classes/coder/proto` | 9 种包类型 | `WKConnectPacket.m`、`WKConnackPacket.m`、`WKSendPacket.m`、`WKSendackPacket.m`、`WKRecvPacket.m`、`WKRecvackPacket.m`、`WKPingPacket.m`、`WKPongPacket.m`、`WKDisconnectPacket.m` |
| `Classes/manager` | 19 个管理器 | `WKConnectionManager.m`（840）、`WKChatManager.m`（1824）、`WKMessageQueueManager`、`WKRetryManager`、`WKReceiptManager`（已读回执）、`WKFlameManager`（阅后即焚）、`WKReminderManager`、`WKReactionManager`、`WKRobotManager`、`WKMediaManager`、`WKSecurityManager.m`（86） |
| `Classes/db` | 15 个表 DB + 迁移 | `WKDB.m`（528）、`WKMessageDB.m`（1170）、`WKConversationDB`、`WKMessageExtraDB`、`WKChannelInfoDB`、`WKChannelMemberDB`、`WKReminderDB`、`WKReactionDB`、`WKRobotDB`、`WKCMDDB`、`WKPinnedMessageDB`、`WKDBMigrationManager` |
| `Classes/model` | 消息/会话/频道模型 | `WKMessage.m`、`WKConversation.m`、`WKChannel.m`、`content/`（消息内容族） |
| `Classes/utils` | 加密与工具 | `WKAESUtil.m`、`WKRSAUtil`、`WKMemoryCache`、`WKNOGeneraterUtil` |
| `Classes/private` | 退避算法等 | `WKBackoff.m`、`arm/`（平台相关，待验证内容） |

### 10.6 WuKongBase（框架层）

除 §3.4 的 Sections 外，`Classes/` 根级还有：

| 文件 | 行数 | 职责 |
|---|---|---|
| `WKApp.m` / `WKApp.h` | 1782 / 282 | 应用单例（见 §3.3） |
| `WKAppConfig.m` / `WKAppConfig.h` | — / 含 `connectURL` 声明（`.h:65`） | 全局配置与默认值 |
| `WKConstant.h` | 89+ | point 字符串常量族 |
| `WuKongBase.h` | 156-159 定义 LLang 宏 |  umbrella header + 宏 |
| `WuKongBaseModule.m` | — | WuKongBase 自身的模块注册 |

---

## 11. 关键文件速查表（附录 B）

按主题索引本次审计的核心引用点：

| 主题 | 文件：行号 |
|---|---|
| 硬编码服务器 | `TangSengDaoDao/AppDelegate.m:19-20` |
| 启动装配 | `TangSengDaoDao/AppDelegate.m:52-91` |
| 首页 block 注入 | `TangSengDaoDao/AppDelegate.m:73-76`；`WuKongBase/Classes/WKApp.h:68` |
| appInit 总装 | `WuKongBase/Classes/WKApp.m:274-441` |
| API 配置（公共头/401） | `WuKongBase/Classes/WKApp.m:524-579` |
| 证书校验关闭 | `WuKongBase/Classes/Services/Base/WKAPIClient.m:43-48` |
| 请求/响应全量日志 | `WuKongBase/Classes/Services/Base/WKAPIClient.m:404-417` |
| token 存储（NSUserDefaults） | `WuKongBase/Classes/Services/Base/WKLoginInfo.m:123-135` |
| 登录请求（明文密码） | `WuKongLogin/Classes/Login/Src/WKLoginVM.m:47-51` |
| 登录数据落地 | `WuKongLogin/Classes/Login/Src/WKLoginVM.m:53-94` |
| SQLCipher 密钥=uid | `WuKongIMiOSSDK/WuKongIMSDK/Classes/db/WKDB.m:73-94` |
| IM 连接（无 TLS） | `WuKongIMiOSSDK/WuKongIMSDK/Classes/manager/WKConnectionManager.m:202-214` |
| CONNECT 明文 token | `WuKongIMiOSSDK/WuKongIMSDK/Classes/coder/proto/WKConnectPacket.m:42-60` |
| DH+AES 协商 | `WuKongIMiOSSDK/WuKongIMSDK/Classes/manager/WKSecurityManager.m:39-83` |
| payload 加解密 | `WuKongIMiOSSDK/WuKongIMSDK/Classes/coder/proto/WKSendPacket.m:42`；`WKRecvPacket.m:60-66` |
| 消息发送主链 | `WuKongBase/Classes/Sections/Conversation/WKConversationContextImpl.m:495-520` → `WuKongIMiOSSDK/.../manager/WKChatManager.m:284-321` |
| 帧编解码 | `WuKongIMiOSSDK/WuKongIMSDK/Classes/coder/WKCoder.m:25-65` |
| JS 桥注册 | `WuKongBase/Classes/Sections/Common/Webview/WKWebViewVC.m:243-247` |
| JS handler 清单 | `WuKongBase/Classes/Sections/Common/Webview/WKWebViewService.m:15-101` |
| WebView 证书盲从 | `WuKongBase/Classes/Sections/Common/Webview/WKWebViewVC.m:392-404` |
| 任意 scheme openURL | `WuKongBase/Classes/Sections/Common/Webview/WKWebViewVC.m:361-369` |
| Bugly 初始化 | `WuKongBase/Classes/WKApp.m:234-248` |
| 日志宏（Release=NSLog） | `WuKongBase/Classes/Services/Logs/WKLogs.h:17-30` |
| deviceToken 上报 | `TangSengDaoDao/AppDelegate.m:132-158` |
| 通知扩展 stub | `NotificationService/NotificationService.m:19-28` |
| ATS 豁免 | `TangSengDaoDao/Info.plist`（NSAppTransportSecurity 段） |
| dev 推送环境 | `TangSengDaoDao/TangSengDaoDao.entitlements` |
| Team ID 硬编码 | `Podfile:6` |
| fork 锁定 commit | `Podfile.lock`（CHECKOUT OPTIONS 段） |

---

## 附：分析覆盖度与遗留待验证项

已完整走读的模块：主 app 全部 4 个 .m、NotificationService、NotificationContent、WKApp、WKAppConfig、WKAPIClient、WKLoginInfo、WKLogs、WKSyncService、WKLoginVM/WKLoginVC/WKLoginModule、WKDB、WKConnectionManager（主流程）、WKChatManager（发送路径）、WKCoder、WKConnectPacket、WKSendPacket、WKRecvPacket、WKSecurityManager、WKAESUtil、WKWebViewVC、WKWebViewService、UIDevice+Utils、Podfile/Podfile.lock/pbxproj/Info.plist/entitlements、WuKongBase.podspec。

遗留待验证项：

1. **E2E（Signal 风格端到端加密）的完整实现链路**：UI 与配置层有迹象（`WKEndToEndEncryptHitContent`、`WKSignalErrorContent`、`rsa_public_key`），密钥交换与会话管理的具体实现未展开审计。
2. Socket.IO-Client-Swift / SocketRocket 的实际调用点未定位（疑为历史遗留）。
3. `Pods/` 内容与 Podfile.lock 的一致性未逐项核对。
4. WKAuthWebViewVC（WuKongLogin 内授权 WebView）未单独审计，疑与 WKWebViewVC 共用模式。
5. WKMessageQueueManager 内部限速循环（`WKMessageQueueManager.m:76,111`）与 WKRetryManager 细节未逐行走读。
6. 服务端推送 payload 是否含明文消息（决定 NotificationService stub 是否为安全问题）——需服务端仓库佐证。
7. WKSDK 的 `WKFlameManager`（阅后即焚）、`WKRobotManager` 未展开。
