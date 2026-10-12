# TangSengDaoDaoAndroid（唐僧叨叨 Android 客户端）深度分析报告

> 分析对象：`/data/code/tsdd-analysis/TangSengDaoDaoAndroid`
> 分析方式：纯静态代码审查（未编译、未运行动态测试）
> 许可证：Apache License 2.0（仓库根 `LICENSE`）
> 报告日期：2026-10-12
> 文档约定：`file:line` 均为已 Read 核实的真实引用；无法完全确认处以"待验证"标注。

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

---

## 1. 仓库概述

### 1.1 定位

本仓库是开源 IM 产品「唐僧叨叨」的 **Android 客户端**，底层通讯能力由「悟空IM」（WuKongIM）提供，IM SDK 以 JitPack 外部依赖 `com.github.WuKongIM:WuKongIMAndroidSDK:1.5.0` 形式引入（`wkbase/build.gradle`，无 SDK 源码，属黑盒）。README 自述："唐僧叨叨基于底层通讯框架悟空IM实现聊天功能……由于demo是模块化开发，开发者可完全按自己的开发习惯进行二次开发"、"100%开源，商业开发无需授权可直接使用"。

功能覆盖：

- 单聊/群聊（README 宣称支持万人群）、文本/图片/语音/名片/合并转发消息；
- emoji、GIF 表情（`robot/` 目录含 RobotGIFAdapter）、Lottie 贴纸（rlottie）；
- 消息链接、手机号、邮箱识别（富文本 span 体系）；
- 通讯录/好友申请/手机通讯录匹配（`POST/GET user/maillist`）；
- 群管理（公告、免打扰、置顶、群昵称、管理员、审核）；
- 全局搜索（用户/聊天记录/以图搜图 `SearchWithImgActivity`）；
- 扫一扫（加好友、登录 PC/Web）；
- 多端在线状态同步（`GET/POST user/online`）、消息已读回执、消息编辑/撤回/点赞（reaction 同步接口）；
- 敏感词/违禁词体系（`message/sync/sensitivewords`、`message/prohibit_words/sync`，本地 `SensitiveWords` 过滤）；
- 五厂商通道离线推送（华为/小米/OPPO/vivo/FCM）；
- 音视频通话入口：仓库内仅有挂钩（`EndpointManager` 的 `rtc_is_calling`、`show_rtc_notification` 端点、`newrtc.mp3` 铃声、`TabActivity` 中 RTC channel 通知），**RTC 实现不在本仓库**，属外挂商业模块（待验证其归属）。

### 1.2 架构意图：薄壳 app + SDK 化功能模块

工程采用「**一个薄壳 app 模块 + 一组 library 功能模块**」的结构：

- `app` 模块仅 5 个源文件（`TSApplication.kt`、`MainActivity.kt`、`AppFrontBackHelper.java` + 2 个测试 stub），包名残留 `com.test.demo2`，`applicationId` 与 `namespace` 在构建期从 `local.properties` 注入（`app/build.gradle:6-27`）。
- 全部业务能力在 `wkbase` / `wkuikit` / `wklogin` / `wkscan` / `wkpush` 五个 library 模块中，命名空间分别为 `com.chat.base`、`com.chat.uikit`、`com.chat.login`、`com.chat.scan`、`com.chat.push`。
- 功能模块间**没有 Gradle 互相依赖**（`wklogin`/`wkscan`/`wkpush`/`wkuikit` 均只 `api project(':wkbase')`），跨模块调用全部通过 `wkbase` 提供的 **EndpointManager 字符串路由**完成（见 §3.2）。

这一结构的意图非常明确：**这套 UIKit 可被第三方 OEM 集成**——第三方只需新建自己的 app 壳模块、在 `local.properties` 写 `applicationId`、运行时配置服务器地址（`api_base_url`）、替换推送厂商 key，即可得到自有品牌 IM App。README 也明确："二次开发时只需要替换对应的 appID 和 appKey 即可"。

但存在两处"半 SDK 化"痕迹：

- `app` 壳源码硬编码 `import com.test.ts.R`（`app/src/main/java/com/test/demo2/TSApplication.kt:38`、`MainActivity.kt:20-21`），`app/build.gradle:22` 的构建期报错文案也明说"请在 local.properties 中设置 applicationId……并在 MainActivity 和 TSApplication 文件中导包设置的包名"——即换包名需**手工同步修改 import**，未做到零改动 OEM。
- `app/google-services.json`（FCM 配置，Firebase 工程 `tsdddemo`）与 `wkpush/PushKeys.java` 中的厂商推送 key 均为唐僧叨叨官方值，OEM 必须替换，否则终端会向官方推送通道注册。

### 1.3 基本信息核实

| 项 | 值 | 出处 |
|---|---|---|
| root 工程名 | TsddClient | `settings.gradle:25` |
| Gradle 模块 | app、wkbase、wkuikit、wklogin、wkscan、wkpush、MyLibs:rlottie、MyLibs:oppopush、MyLibs:vivopush | `settings.gradle:26-34` |
| AGP / Kotlin / Java | AGP 8.13.0 / Kotlin 2.2.0 / Java 17 | 根 `build.gradle:4,15,19`；各模块 `compileOptions` |
| minSdk / targetSdk / compileSdk / buildTools | 24 / 34 / 35 / 35.0.0 | 根 `build.gradle:41-45` |
| Kotlin / Java 文件数 | 123 / 448（含测试 stub） | 全仓库统计；分模块见 §8.2 |
| 总代码量 | 约 9.24 万行（Java+Kotlin，不含 MyLibs 二进制） | `wc -l` 统计 |
| 测试 | 仅 AS 模板 stub（`ExampleUnitTest`、`ExampleInstrumentedTest`），无真实用例 | 如 `app/src/test/java/com/xinbida/tsdd/demo/ExampleUnitTest.kt`（验证 2+2=4） |
| 后端协议 | HTTP REST `{api_base_url}/v1/` + WKIM TCP 长连接 | §5 |

---

## 2. 技术栈与依赖

### 2.1 模块 × 依赖矩阵

| 依赖 | app | wkbase | wkuikit | wklogin | wkscan | wkpush |
|---|---|---|---|---|---|---|
| 模块依赖 | 全部 | — | wkbase | wkbase | wkbase | wkbase |
| WuKongIMAndroidSDK 1.5.0（JitPack） | | api | (经 wkbase) | | | |
| Retrofit 3.0.0 + adapter-rxjava3 3.0.0 | | api | | | | |
| OkHttp 5.1.0 | | api | | | | |
| FastJSON 1.2.83（阿里） | | api | | | | |
| converter-gson 3.0.0（排除 gson 传递依赖） | | api | | | | |
| RxJava3 3.1.11 / RxAndroid 3.0.2 | | api | | | | |
| Glide 5.0.5（JitPack）+ glide-transformations 4.3.0 | | api | | | | |
| PictureSelector v3.11.2 + compress v3.11.2 | | api/impl | | | | |
| gsyvideoplayer 11.3.0（排除 rtmp-client） | | api | | | | |
| jsoup 1.21.2 | | api | | | | |
| XPopup 2.9.19（JitPack） | | api | | | | |
| BaseRecyclerViewAdapterHelper 3.0.14 | | api | | | | |
| SmartRefreshLayout kernel/header 2.1.1 | | api | | | | |
| PanelSwitchHelper v1.5.12（JitPack） | | api | | | | |
| android-image-cropper 4.6.0 | | impl | | | | |
| emoji2 / emoji2-views 1.6.0 | | api | | | | |
| kotlinx-coroutines-core 1.10.2 | | api | | | | |
| MyLibs:rlottie（本地 AAR） | | api | | | | |
| zxing-lite 3.2.0（JitPack） | | | | | impl | |
| 华为 HMS push 6.1.0.300 | | | | | | impl |
| MiPush_SDK_Client_3_7_5.jar（本地 jar） | | | | | | impl |
| MyLibs:oppopush（push-3.0.0.aar） | | | | | | impl |
| MyLibs:vivopush（vivo_pushSDK_v3.0.0.4_484.aar） | | | | | | impl |
| Firebase BOM 32.8.1（analytics + messaging） | | | | | | impl |
| gson 2.8.9 / commons-codec 1.11（OPPO SDK 需要） | | | | | | impl |
| google-services 插件 4.4.2 | 插件 | | | | | 插件 |
| 华为 agcp 1.6.0.300（buildscript classpath） | | | | | | |

仓库源（`settings.gradle:11-21`、根 `build.gradle:5-13`）：google、mavenCentral、**jcenter（已停服，仅靠缓存）**、**jitpack.io**、华为 developer repo、maven.google；buildscript 还声明了 `https://mvnrepository.com`（非标准制品仓库，实际不承载制品）。

### 2.2 供应链性质评估

- **IM SDK 黑盒（中-高风险）**：`com.github.WuKongIM:WuKongIMAndroidSDK:1.5.0` 无源码随仓，长连接协议、消息加解密（`cmdManager.rsaPublicKey` 表明存在 RSA 握手）、本地消息库结构均不可审。SDK 与后端 WuKongIM 服务端配套，安全性取决于外部项目。
- **JitPack 依赖（中风险）**：WuKongIMAndroidSDK、XPopup、glide（JitPack 版 5.0.5）、PanelSwitchHelper、zxing-lite 均经 JitPack 按 git tag 即时构建分发，无法验证制品与源码一致性；`settings.gradle` 使用 `RepositoriesMode.FAIL_ON_PROJECT_REPOS` 但未配置依赖校验（dependency verification）与签名检查。
- **本地 AAR/JAR（较高风险）**：
  - `MyLibs/rlottie/lottie_release.aar`（Telegram rlottie 裁剪版，提供 `org.telegram.lottiejson.LottieUtils`、`org.telegram.ui.Components.RLottieImageView`，`MyLibs/rlottie/build.gradle` 仅一行 `artifacts.add`）；目录内还残留未引用的 `lottiev1.aar`；
  - `MyLibs/oppopush/push-3.0.0.aar`、`MyLibs/vivopush/vivo_pushSDK_v3.0.0.4_484.aar`、`wkpush/libs/MiPush_SDK_Client_3_7_5.jar` 均为厂商推送 SDK 二进制拷贝，**版本固化、无更新机制、无来源校验记录**。
- **FastJSON 1.2.83（重点风险）**：`wkbase/build.gradle` 中 `api 'com.alibaba:fastjson:1.2.83'`。1.2.83 是 fastjson 1.2.x 的最后版本（2021），该系列历史上存在一长串 autotype 反序列化绕过 CVE（1.2.24/1.2.47/1.2.68/1.2.80 等）并已停止维护。本仓库所有 Retrofit 响应都经 `FastJsonResponseBodyConverter.kt:13` 的 `JSON.parseObject(value.string(), type)` 反序列化。按泛型 Type 解析时 fastjson 默认不开启 autotype，直接 RCE 面较小，但实体中存在 `JSONObject` 松散字段及 `WKUIKitApplication.java:158` 等直接解析处，叠加 §7.1 的 MITM 能力后风险被放大。官方建议迁移 fastjson2。详见 §7.6。

### 2.3 依赖管理其他观察

- 根 `build.gradle:27-37` 用 `resolutionStrategy.force` 全局强制 Kotlin stdlib 2.2.0 与 CameraX 1.5.3（仓库中未见 CameraX 直接依赖，疑为遗留/防御性配置，待验证）。
- `wkbase` 同时声明 `converter-gson` 且 `exclude group: 'com.google.code.gson'`——实际序列化走 FastJSON 自定义 converter（`RetrofitUtils.java:32`），Gson 仅被 `WKConfig`（`WKConfig.java:69,77,86,94`）、`WKWebViewActivity` 等零星使用。序列化栈双轨并存。
- `compileSdk 35` 与 `targetSdk 34` 不一致；`README.md` 徽标仍写 compileSdkVersion 33 / minSdk 23，文档已过期。
- `android.nonTransitiveRClass=false`、`android.nonFinalResIds=false`（`gradle.properties`）——关闭了新式 R 类优化，牺牲构建速度换旧代码兼容。

---

## 3. 目录结构与模块架构

### 3.1 模块依赖图

```mermaid
graph TD
    subgraph 壳
        APP["app<br/>(com.test.demo2 / applicationId 构建期注入)"]
    end
    subgraph 功能模块
        LOGIN["wklogin<br/>com.chat.login"]
        UIKIT["wkuikit<br/>com.chat.uikit"]
        SCAN["wkscan<br/>com.chat.scan"]
        PUSH["wkpush<br/>com.chat.push"]
    end
    subgraph 基础层
        BASE["wkbase<br/>com.chat.base"]
        WKIMSDK["WuKongIMAndroidSDK 1.5.0<br/>(JitPack 外部依赖, 无源码)"]
    end
    subgraph 本地二进制
        RLOTTIE["MyLibs:rlottie<br/>(lottie_release.aar)"]
        OPPO["MyLibs:oppopush<br/>(push-3.0.0.aar)"]
        VIVO["MyLibs:vivopush<br/>(vivo_pushSDK_v3.0.0.4_484.aar)"]
    end

    APP --> LOGIN
    APP --> UIKIT
    APP --> SCAN
    APP --> PUSH
    APP --> BASE
    LOGIN --> BASE
    UIKIT --> BASE
    SCAN --> BASE
    PUSH --> BASE
    BASE --> WKIMSDK
    BASE --> RLOTTIE
    PUSH --> OPPO
    PUSH --> VIVO

    LOGIN -. "EndpointManager 字符串路由" .-> UIKIT
    SCAN -. "EndpointManager" .-> LOGIN
    PUSH -. "EndpointManager" .-> BASE
```

注：`wklogin`/`wkuikit`/`wkscan`/`wkpush` 之间**没有 Gradle 依赖**，虚线为运行时经 EndpointManager 的隐式调用（详见 §3.2）。

### 3.2 EndpointManager 模块通信机制

核心文件：`wkbase/src/main/java/com/chat/base/endpoint/EndpointManager.java`

- **数据结构**：`ConcurrentHashMap<String /*category*/, List<Endpoint>>`（`EndpointManager.java:25`）。每个 `Endpoint` 含 `sid`（字符串 ID）、`category`（分类）、`sort`（排序）、`iHandler`（`EndpointHandler` 函数式接口，入参/出参均为 `Object`）。
- **注册**：`setMethod(sid, handler)`（`:40`）、`setMethod(sid, category, handler)`（`:44`）、`setMethod(sid, category, sort, handler)`（`:48`）。
- **调用**：
  - `invoke(sid, param)`（`:72-90`）：**全表扫描**所有 category，从每个列表尾部向前找第一个 sid 匹配者（即同 sid 后注册优先），返回 handler 结果；找不到时返回 null（调用方需判空，如 `TSApplication.kt:154-158`）。
  - `invokes(category, param)`（`:92-110`）：按 category 取列表、`Collections.sort` 按 sort 排序后逐个调用，收集非 null 结果——用于"一个槽位多个模块挂 UI 项"的场景（tab 菜单、聊天面板、消息长按菜单、DB 调试菜单）。
- **预定义契约**：category 集中于 `EndpointCategory.java`（`tabMenus="tab_menus"`、`wkScan="wk_scan"`、`msgConfig="msg_config"`、`wkChatPopupItem`、`loginMenus`、`wkDBMenus`、`chatFunction`、`personalCenter` 等）；部分 sid 集中于 `EndpointSID.java`（`chatView="chat_show_chat_view"`、`sendMessage="send_message"` 等）。**但大量 sid 是散落各处的裸字符串**（如 `"show_tab_home"`、`"rtc_is_calling"`、`"chow_check_lock_screen_pwd"`、`"uikit_sql"`），无集中登记，属隐式契约。
- **典型用例**：
  - `TSApplication.kt:172-193` 注册 `main_show_home_view`、`show_tab_home`、`play_new_msg_Media`；
  - `TSApplication.kt:142-144` 前台时调用 `chow_check_lock_screen_pwd`（锁屏密码模块，本仓库未见注册方，待验证）；`:154` 调用 `rtc_is_calling`（RTC 外挂模块）；
  - `WKLoginApplication.java:42-60` 向 `EndpointCategory.wkScan` 注册扫码结果处理器（`loginConfirm` 类型 → 打开 PC 登录确认页）；
  - `WKScanApplication.java:42-52` 注册 `create_qrcode`/`parse_qrcode` 供其他模块调用；
  - `WKPushApplication.java:64` 向 `loginMenus` 注册"登录成功后初始化推送"；`:185-190` 注册 `push_update_device_badge`；
  - `WKUIKitApplication.java:215-223` 注册各类消息的 `msgConfig` 长按菜单配置；`:220` 注册 `uikit_sql` 数据库调试菜单。

**部分端点注册/调用对照**（静态梳理，完整清单需动态验证）：

| sid / category | 注册方 | 调用方 | 用途 |
|---|---|---|---|
| `main_show_home_view` | `TSApplication.kt:172` | 外部/推送点击（待验证） | 回主界面并携带 from |
| `show_tab_home` | `TSApplication.kt:182` | 登录成功等 | 打开 TabActivity |
| `play_new_msg_Media` | `TSApplication.kt:189` | wkuikit 新消息到达 | 播放提示音 |
| `EndpointCategory.wkScan` | `WKLoginApplication.java:42` | wkscan 扫码结果分发 | 扫码登录确认路由 |
| `wk_scan_show` | `WKScanApplication.java:36` | 其他模块 | 打开扫码页 |
| `create_qrcode` / `parse_qrcode` | `WKScanApplication.java:47,52` | wkuikit/wklogin | 二维码生成与解析 |
| `EndpointCategory.loginMenus` | `WKPushApplication.java:64` | 登录成功回调 | 登录后初始化推送 |
| `push_update_device_badge` | `WKPushApplication.java:185` | `WKUIKitApplication.java:192` | 同步桌面角标 |
| `msgConfig + 消息类型` | `WKUIKitApplication.java:215-223` | `WKChatBaseProvider` 长按 | 消息长按菜单配置 |
| `EndpointSID.sendMessage` | 各模块可挂 | `WKSendMsgUtils.java:47` | 发消息前广播 |
| `chat_show_chat_view`（EndpointSID.chatView） | wkuikit（注册点待验证） | `WKWebViewActivity.java:249` 等 | 打开聊天页 |
| `rtc_is_calling` | **本仓库未见注册方**（外挂 RTC 模块） | `TSApplication.kt:154` | 退后台时判断是否通话中 |
| `chow_check_lock_screen_pwd` | **本仓库未见注册方**（外挂锁屏模块） | `TSApplication.kt:142-143` | 回前台检查锁屏密码 |
| `update_base_url` | **全仓库未见注册方（死调用）** | `WKLoginActivity.java:180,186`、`ThirdLoginActivity.kt:69` | 修改服务器地址后刷新网络层 |
| `uikit_sql`（category `wkDBMenus`） | `WKUIKitApplication.java:220` | 设置-错误日志/调试页（待验证） | 数据库调试入口 |
| `show_open_notification_dialog` | `WKPushApplication.java:164` | `TabActivity.java:99,107` | 引导开启通知权限 |
| `play_video` | `WKBaseApplication.java:94-110` | 聊天视频消息 | 视频播放页 |

**评价**：这是一个字符串路由的服务定位器（Service Locator）。优点是解耦彻底（功能模块互不依赖、可插拔），代价是：

1. 无编译期检查（sid 拼写错误静默失败）；
2. 无参数类型安全（`Object` 强转，错误类型运行期才炸）；
3. `invoke` 为 O(端点总数) 线性扫描；
4. 同 sid 覆盖语义隐晦（后注册先匹配）；
5. **存在死调用**：`wklogin` 中 `EndpointManager.getInstance().invoke("update_base_url", ...)`（`WKLoginActivity.java:180,186`、`ThirdLoginActivity.kt:69`）全仓库搜不到对应的 `setMethod("update_base_url", ...)` 注册点（待验证是否在外部模块），其直接后果是**修改服务器地址后 Retrofit 单例不重建**——`RetrofitUtils.resetRetrofit()` 定义于 `RetrofitUtils.java:40-42` 但无人调用，baseUrl 缓存在 `RetrofitUtils.java:27-34` 的单例里，需杀进程重启才生效。

### 3.3 各模块职责与包结构

| 模块 | namespace | 职责 | 关键内容 |
|---|---|---|---|
| app | （local.properties 注入） | 薄壳：进程初始化、隐私协议、登录/主界面路由、前后台管理、通知渠道 | `TSApplication.kt`、`MainActivity.kt`、`AppFrontBackHelper.java` |
| wkbase | com.chat.base | 网络层（Retrofit/OkHttp/FastJSON converter）、配置与 SP、Endpoint 框架、SQLite 封装、基础 UI 组件/主题/多语言、WebView 与 JSBridge、emoji、Glide、上传下载、崩溃收集 | `WKBaseApplication.java`、`net/`(27 文件)、`endpoint/`(60)、`ui/`(47)、`views/`(41)、`utils/`(40)、`msgitem/`(14) |
| wkuikit | com.chat.uikit | 主功能 UI：Tab 导航、会话列表、聊天页、通讯录、群组、机器人、搜索、设置、"我的"、消息渲染体系、IM SDK 初始化与事件桥接 | `TabActivity.java`、`chat/`(43)、`group/`(21)、`search/`(19)、`view/`(19)、`WKUIKitApplication.java`、`message/MsgModel.java` |
| wklogin | com.chat.login | 登录/注册/找回密码、短信验证码、第三方授权登录（gitee/github）、PC/Web 扫码登录确认、服务器地址设置入口 | `ui/WKLoginActivity.java`、`service/LoginModel.java`、`service/LoginService.java` |
| wkscan | com.chat.scan | 二维码扫描（zxing-lite）、扫码结果分发（经 `EndpointCategory.wkScan`）、二维码生成/解析 | `WKScanActivity.java`、`ScanUtils.java` |
| wkpush | com.chat.push | 五通道离线推送、厂商 ROM 检测、token 上报、角标同步 | `WKPushApplication.java`、`OsUtils.java`、`push/`（5 个 Receiver/Service）、`service/PushModel.java` |
| MyLibs:rlottie | — | Telegram rlottie 裁剪 AAR：Lottie/贴纸渲染（含 native .so） | `lottie_release.aar` |
| MyLibs:oppopush / MyLibs:vivopush | — | OPPO/vivo 厂商推送 SDK 二进制 | aar |

wkuikit 包结构（`wkuikit/src/main/java/com/chat/uikit/`）：`chat`（聊天页/面板/消息 model/Provider）、`contacts`（通讯录）、`group`（群组）、`message`（消息同步 model/service）、`robot`（机器人/inline query/GIF）、`search`（搜索）、`setting`（设置/主题/语言/字号/错误日志）、`user`（用户资料）、`fragment`（ChatFragment 会话列表/ContactsFragment/MyFragment）、`db`（WKContactsDB）、`crypto`（Signal 密钥服务）、`enity`（实体，拼写如此）。

---

## 4. 核心流程走读

### 4.a 启动与服务器地址配置流

```mermaid
sequenceDiagram
    participant App as TSApplication
    participant SP as SharedPreferences
    participant Base as WKBaseApplication
    participant Main as MainActivity
    participant Login as WKLoginActivity
    participant Tab as TabActivity

    App->>App: onCreate() 仅主进程 initAll() (TSApplication.kt:44-50)
    App->>Base: init(packageName, app) (WKBaseApplication.java:69)
    App->>SP: getSP("api_base_url") (TSApplication.kt:110)
    alt api_base_url 为空
        App->>App: throw IllegalStateException (TSApplication.kt:114)
    else 已配置
        App->>App: WKApiConfig.initBaseURLIncludeIP(url) (TSApplication.kt:116)
    end
    App->>App: 初始化 login/scan/uikit/push 模块 (TSApplication.kt:101-104)
    Main->>SP: show_agreement_dialog (MainActivity.kt:32-33)
    alt 首启
        Main->>Main: 隐私协议弹窗 (MainActivity.kt:62-114)
    end
    Main->>Main: gotoApp() (MainActivity.kt:39-60)
    alt 无 token
        Main->>Login: 登录页
    else 有 token 但无用户名
        Main->>Main: PerfectUserInfoActivity
    else 有 token 但无 IM RSA 公钥
        Main->>Login: 重新登录/握手 (MainActivity.kt:44-49)
    else
        Main->>Tab: 主界面
    end
```

调用链要点：

1. `TSApplication.onCreate()`（`app/src/main/java/com/test/demo2/TSApplication.kt:42-74`）：取进程名，仅主进程执行 `initAll()`（`:95-107`）；注册 ActivityLifecycle 回调跟踪 `currentActivity`（`ActManagerUtils`）。
2. `initAll()` 依次：多语言 → `WKBaseApplication.getInstance().init(packageName, this)`（`wkbase/.../WKBaseApplication.java:69`，初始化 Glide 替换 OkHttp 加载器、缓存目录、emoji、rlottie、CrashHandler）→ `initApi()` → `WKLoginApplication.init` → `WKScanApplication.init` → `WKUIKitApplication.init`（内部 `initIM()` 初始化 WKIM SDK，见 §4.c）→ `WKPushApplication.init` → 前后台监听（`addAppFrontBack`，`:135-168`）。
3. **`initApi()`（`TSApplication.kt:109-118`）**：从 SharedPreferences 读 `api_base_url`；**为空则直接 `throw IllegalStateException("api_base_url 未设置，请先配置接口地址")`（:114）——首次安装若未预埋地址，App 在 Application 阶段崩溃**；非空调用 `WKApiConfig.initBaseURLIncludeIP(apiURL)`，拼出 `baseUrl = apiURL + "/v1/"`、`baseWebUrl = apiURL + "/web/"`（`wkbase/.../config/WKApiConfig.java:20-23`）。**服务器地址不编译进包**，属 OEM/私有化部署设计。
4. `api_base_url` 的运行时写入入口（均在登录界面，且受后端开关 `can_modify_api_url` 控制，`GET common/appconfig`）：
   - `WKLoginActivity.java:169-184`：点击 baseUrlTv 弹输入框，**未以 http 开头时自动补 `http://`（:176）**，写 SP 后 invoke `update_base_url`（无注册方，见 §3.2）；
   - `ThirdLoginActivity.kt:50-73`：同样逻辑（:66 补 `http://`）；
   - 重置按钮清空 `api_base_url`（`WKLoginActivity.java:185-187`）。
5. `MainActivity.initView()`（`MainActivity.kt:30-37`）：读 SP `show_agreement_dialog`（**默认 true**——`WKSharedPreferencesUtil.getBoolean(key)` 缺省值 true，`WKSharedPreferencesUtil.java:87`）→ 首启显示隐私协议弹窗（`:62-114`，用户协议链接 `baseWebUrl + "user_agreement.html"`）；同意后重置标记并重新 init `WKBaseApplication`（`:103-108`，因其 init 在协议未同意时提前 return，`WKBaseApplication.java:75-78`）。
6. `gotoApp()`（`MainActivity.kt:39-60`）路由规则见时序图。

### 4.b 登录与 token 管理

```mermaid
sequenceDiagram
    participant UI as WKLoginActivity
    participant P as LoginPresenter
    participant M as LoginModel
    participant API as POST /v1/user/login
    participant Cfg as WKConfig(SP)
    participant IM as WKIM SDK

    UI->>P: login(name, pwd) (LoginPresenter.java:30)
    P->>M: loginApp(name, pwd) (LoginModel.java:52)
    M->>API: {username, password(明文), device{...}} (LoginModel.java:53-60)
    API-->>M: UserInfoEntity{token, im_token, uid, ...}
    M->>Cfg: saveLoginInfo (LoginModel.java:363-371)
    Note over Cfg: SP 明文: wk_token / wk_im_token / wk_uid / user_info
    M-->>UI: loginResult → TabActivity
    UI->>IM: init(context, uid, imToken) (WKUIKitApplication.java:179)
```

1. 登录页 `WKLoginActivity` → `LoginPresenter.login(name, pwd)`（`wklogin/.../LoginPresenter.java:30-46`）→ `LoginModel.loginApp`（`LoginModel.java:52-87`）。
2. **请求体**：`{"username": name, "password": pwd, "device": {device_id, device_name, device_model}}`（`LoginModel.java:53-60`）。**密码客户端侧不做 MD5/哈希/加密，直接明文放入 JSON**，依赖传输层保护（而传输层已被 §7.1 架空）；device_id 取 `ANDROID_ID`，为空时生成随机 UUID 存 SP（`WKConstants.java:48-61`）。接口 `POST user/login`（`LoginService.java:26-27`）。
3. 成功后 `saveLoginInfo`（`LoginModel.java:363-371`）：
   - `WKConfig.setToken(userInfo.token)` → SP 键 `wk_token`（`WKConfig.java:34-40`）；
   - `setImToken(im_token)`（缺省回退 token）→ SP `wk_im_token`（`:42-48`）；
   - `setUid` → SP `wk_uid`（`:26-32`）；
   - `saveUserInfo` 将整个 UserInfoEntity Gson 序列化存 SP `user_info`（`:85-88`）。
   - **全部为 SharedPreferences 明文存储**（文件名 `wkSharedPreferences`，MODE_PRIVATE，`WKSharedPreferencesUtil.java:22-23`）。
4. **token 注入**：每个请求经 `CommonRequestParamInterceptor`（`wkbase/.../net/CommonRequestParamInterceptor.java:40-49`）统一加 header：`token`、`model`、`os=Android`、`appid=wukongchat`（硬编码于 `WKBaseApplication.java:52`）、`version`、`package`。
5. **自动登录**：下次启动 `MainActivity.gotoApp()` 凭 SP 中 token 直接进 `TabActivity`（§4.a）；`WKUIKitApplication.initIM()`（`WKUIKitApplication.java:170-182`）用 SP 中 uid + imToken 调 `WKIM.getInstance().init(context, uid, imToken)`。
6. **其他登录方式**：
   - 短信验证码登录：`POST user/sms/login_check_phone` + `POST user/login/check_phone`（`LoginService.java:51-55`）；
   - 第三方授权（gitee/github）：`ThirdLoginActivity.kt` + `GET user/thirdlogin/authcode`、`GET user/thirdlogin/authstatus`（轮询）；
   - PC/Web 扫码登录：扫出 `loginConfirm` JSON → `WKWebLoginConfirmActivity` → `GET user/grant_login`（`LoginService.java:29-30`、`WKLoginApplication.java:42-60`）；`user/pc/quit` 踢 PC 下线。
7. **二次验证**：code==110 时要求短信二次校验（`LoginPresenter.java:37-38`）。
8. **登出**：`PushModel.unRegisterDeviceToken`（`DELETE user/device_token`）+ `WKConfig.clearInfo()`（`WKConfig.java:58-66`）清 token/uid，并断开 IM（`WKUIKitApplication.java:555`）。

### 4.c 消息收发

**连接建立与生命周期：**

1. `WKUIKitApplication.startChat()`（`WKUIKitApplication.java:184-189`）→ `WKIM.getInstance().getConnectionManager().connection()`；`stopConn()`（`:191-194`）→ `disconnect(false)`。
2. 前后台切换由 `TSApplication.addAppFrontBack()`（`TSApplication.kt:135-168`）驱动（Activity 计数法，`AppFrontBackHelper.java` 的 `activityStartCount` 0→1 判定回前台）：
   - 回前台：有 token 则 `initIMListener()` + `startChat()` + 同步在线用户（`UserModel.getOnlineUsers`）；若 `disconnect` 标记还延时 1s 调 `chow_check_lock_screen_pwd` 端点（锁屏密码，外挂模块）；
   - 退后台：无 RTC 通话（`rtc_is_calling` 端点）时 `stopConn()`，记录 `lock_start_time`。
3. 保活：`AliveJobService`（`wkuikit/.../AliveJobService.java:26-60`）JobScheduler 周期任务中再次 `connection()`；7.0+ 采用 1s 最小延迟自续期方式（`:37-42`）。
4. **长连接地址来源**：WKIM SDK 通过 `addOnGetIpAndPortListener` 回调索取 IP/端口（`wkuikit/.../chat/manager/WKIMUtils.java:148`）→ `MsgModel.getChatIp`（`message/MsgModel.java:263-278`）→ `GET users/{uid}/im`（`MsgService.java:41-42`）→ 解析响应 `tcp_addr`（`ip:port` 格式）。即 **HTTP API 先路由、TCP 长连地址由后端动态下发**，支持接入点调度。
5. RSA 公钥握手：`MainActivity.kt:44-46` 检查 `cmdManager.rsaPublicKey`，表明 SDK 连接层存在 RSA 密钥交换（细节在 SDK 黑盒内，待验证）。

**收消息路径：**

```mermaid
sequenceDiagram
    participant TCP as WKIM 长连接
    participant SDK as WKIM SDK
    participant Hook as WKIMUtils 监听器
    participant HTTP as MsgModel(HTTP同步)
    participant UI as ChatFragment/ChatActivity

    TCP-->>SDK: 消息帧
    SDK->>Hook: addMessageStoreBeforeIntercept (WKIMUtils.java:150-165)
    alt 截屏提示且频道已关闭提示
        Hook-->>SDK: return false(拦截存库)
    else
        Hook-->>SDK: 放行存库
    end
    SDK->>HTTP: sync 监听器回调(会话/消息/extra)
    HTTP->>HTTP: POST message/sync 等拉取 (MsgService.java:50-88)
    HTTP-->>SDK: 回写同步结果 + syncack
    SDK-->>UI: 刷新会话列表/聊天页
    UI->>UI: 播放提示音 play_new_msg_Media (TSApplication.kt:189-192)
```

1. SDK 长连接收消息 → 存库前拦截 `addMessageStoreBeforeIntercept`（`WKIMUtils.java:150-165`：截屏提示类消息按频道设置 `WKChannelExtras.screenshot` 决定是否拦截）。
2. 历史/离线数据走 HTTP 同步接口，由 `MsgModel` 桥接到 SDK 各 sync 监听器：`POST message/sync`（`MsgService.java:50-51`）、`POST conversation/sync`（会话）、`POST message/channel/sync`（频道内）、`POST message/extra/sync`（消息扩展/点赞）、`POST message/reminder/sync`（提醒）、`POST conversation/extra/sync`；同步确认 `POST message/syncack/{last_message_seq}`、`POST conversation/syncack`。
3. 新消息提示音经端点 `play_new_msg_Media` 播放 `R.raw.newmsg`（`TSApplication.kt:189-192`）。
4. 会话列表（`fragment/ChatFragment.java`，1082 行）监听连接状态（`ChatFragment.java:468` 注册、`911` 注销）并展示未读红点；聊天页（`chat/ChatActivity.java`，2668 行）同样按 channelId 监听连接状态（`ChatActivity.java:821`），处理离线拉取、消息状态刷新、已读回执等场景。

**消息类型与渲染体系：**

- 注册机制：`WKMsgItemViewManager.addChatItemViewProvider(type, provider)` 绑定"消息类型 → View Provider"；基类 `wkbase/msgitem/WKChatBaseProvider.kt`（1621 行，处理气泡/头像/间距/长按）；应用侧注册集中在 `WKUIKitApplication.initKitModuleListener()`（`WKUIKitApplication.java:196-213`）。
- 自定义消息内容模型注册进 SDK：`WKIM.getInstance().getMsgManager().registerContentMsg(WKCardContent.class)`（名片，`type = WKMsgContentType.WK_CARD`，`chat/msgmodel/WKCardContent.java:21`）、`registerContentMsg(WKMultiForwardContent.class)`（合并转发）（`WKUIKitApplication.java:198-201`）。
- **类型清单**（`wkbase/msgitem/WKContentType.java`，继承 SDK 的 `WKMsgContentType`；SDK 侧类型数值待验证）：
  - SDK 内置：`WK_TEXT`、`WK_IMAGE`、`WK_VOICE`、`WK_VIDEO`、`WK_FILE`、`WK_LOCATION`、`WK_GIF`、`WK_STICKER`、`WK_CARD`、`WK_MULTIPLE_FORWARD` 等；
  - 本仓库扩展（`WKContentType.java:12-49`）：系统消息 `systemMsg=0`、富文本 `richText=14`、截屏提示 `screenshot=20`、群事件 1002/1003/1005/1006/1008/1009（加人/减人/群系统消息/撤回/新管理员/审核成员，`isSystemMsg` 判定 1000-2000，`:51-53`）；
  - 本地展示类型（负数，`isLocalMsg` 判定 ≤0，`:55-57`）：`msgPromptNewMsg=-1`、`msgPromptTime=-2`、`unknown_msg=-3`、`typing=-4`、`revoke=-5`、`loading=-6`、`videoCallGroup=-7`、`noRelation=-9`（非好友提示）、`sensitiveWordsTips=-10`、`emptyView=-12`、`spanEmptyView=-13`；
  - 支持通知栏展示的类型区间：`WK_TEXT ~ richText`（`isSupportNotification`，`:59-61`）。
- Provider 清单：`WKTextProvider`（1083 行，含链接/手机号/邮箱/表情 span 与选择复制）、`WKImageProvider`、`WKVoiceProvider`、`WKCardProvider`、`WKMultiForwardProvider`、`WKSensitiveWordsProvider`、`WKNoRelationProvider`、`WKPromptNewMsgProvider`、`WKTypingProvider`、`WKSystemProvider`、`WKRevokeProvider`、`WKUnknownProvider`、`LoadingProvider` 等（`wkbase/msgitem/`、`wkuikit/chat/provider/`）。
- 消息长按菜单经 `EndpointCategory.msgConfig + type` 与 `wkChatPopupItem` 端点动态组装（`WKUIKitApplication.java:215-238`，复制/转发等菜单项按类型注册）。

**发消息路径：**

```mermaid
sequenceDiagram
    participant Panel as ChatPanelManager
    participant Send as WKSendMsgUtils
    participant EP as EndpointManager
    participant SDK as WKIM SDK
    participant HTTP as WKUploader(HTTP)
    participant TCP as 长连接

    Panel->>Send: sendMessage(wkMsg) (WKSendMsgUtils.java:40)
    Send->>EP: invokes(sendMessage, menu) (WKSendMsgUtils.java:47)
    alt 含附件(图/语音/视频/文件)
        SDK-->>Send: onUploadAttach 回调 (WKIMUtils.java:166-167)
        Send->>HTTP: uploadChatAttachment (WKSendMsgUtils.java:75+)
        HTTP-->>SDK: 返回文件URL
    end
    Send->>SDK: msgManager.sendWithOptions (WKSendMsgUtils.java:48)
    SDK->>TCP: 发送消息帧
    TCP-->>SDK: 发送回执/同步
```

1. 聊天面板 `ChatPanelManager.kt`（2710 行：文本/语音/图片/视频/文件/表情/GIF/机器人等输入源）构造消息内容模型；
2. 统一出口 `WKSendMsgUtils.sendMessage(wkMsg)`（`chat/manager/WKSendMsgUtils.java:40-49`）：先 `EndpointManager.invokes(EndpointSID.sendMessage, ...)` 广播（供外挂模块感知/拦截），再 `WKIM.getInstance().getMsgManager().sendWithOptions(content, channel, options)` 交 SDK 经长连接发送；
3. 附件上传：含附件消息（图/GIF/语音/位置封面/文件，`:80-82` 类型判断）由 SDK 回调 `addOnUploadAttachListener`（注册于 `WKIMUtils.java:166-167`）→ `WKSendMsgUtils.uploadChatAttachment` → `wkbase/net/ud/WKUploader` 先 HTTP 上传取 URL 再发消息体；
4. 批量转发 `sendMessages` 以 150ms 定时器逐条发送（`:51-69`）；
5. 输入中状态 `POST message/typing`（`MsgModel.java:284-299`）；语音已读 `PUT message/voicereaded`；撤回 `POST message/revoke`；编辑 `POST message/edit`。

### 4.d 离线推送（五通道）

```mermaid
sequenceDiagram
    participant App as WKPushApplication
    participant GMS as GoogleApiAvailability
    participant OS as OsUtils(ROM检测)
    participant Vendor as 厂商SDK(HMS/MI/OPPO/vivo/FCM)
    participant Backend as POST /v1/user/device_token

    App->>App: initPush() (WKPushApplication.java:67)
    App->>GMS: isGooglePlayServicesAvailable (WKPushApplication.java:194)
    alt GMS 可用
        App->>Vendor: FCM getToken (WKPushApplication.java:197)
        Vendor-->>Backend: registerDeviceToken(token, bid, "FIREBASE")
    else GMS 不可用且已登录
        App->>OS: isEmui/isMiui/isOppo/isVivo (OsUtils.java:43-55)
        OS-->>App: 按 ROM 选通道
        App->>Vendor: 对应厂商注册 (WKPushApplication.java:209-219)
        Vendor-->>Backend: registerDeviceToken(token, bid, HMS/MI/OPPO/VIVO)
    end
```

1. 入口：`WKPushApplication.init(pushBundleID, context)`（`wkpush/.../WKPushApplication.java:59-65`）→ `initPush()`（`:67-84`）→ Firebase 初始化 + `getPushToken()`。
2. **通道选择逻辑（`getPushToken()`，:193-220）**：先查 GMS——
   - GMS 可用 → FCM：`FirebaseMessaging.getInstance().getToken()` 成功后上报（:206）；
   - GMS 不可用且已登录 → 按 ROM 检测（`OsUtils.isEmui()/isMiui()/isOppo()/isVivo()`，基于 `ro.build.version.emui`、`ro.build.version.opporom` 等系统属性，`OsUtils.java:35-55`）依次初始化华为/小米/OPPO/vivo（:209-219）。
3. **各通道 token 获取与上报**：
   - 华为：`getHuaWeiToken()`（:86-101）`HmsInstanceId.getToken(PushKeys.huaweiAPPID, "HCM")`；刷新回调 `HuaweiHmsMessageService.onNewToken`（`push/HuaweiHmsMessageService.java:20-24`）；
   - 小米：`MiPushClient.registerPush(context, xiaoMiAppID, xiaoMiAppKey)`（:103-105）；注册结果 `XiaoMiMessageReceiver.java:80,117` 上报；
   - OPPO：`HeytapPushManager.register(..., oppoAppKey, oppoAppSecret, callback)`（:107-144）；
   - vivo：`PushClient.turnOnPush` 后 `getRegId()`（:146-161），`VivoPushMessageReceiverImpl.java:18` 上报；
   - FCM：`WKFirebaseMessagingService.onNewToken`（`push/WKFirebaseMessagingService.java:16-22`）。
4. **token 上报后端**：`PushModel.registerDeviceToken(token, bundle_id, device_type)`（`service/PushModel.java:39-72`）：device_type 为空时按 ROM 推断（HMS/MI/OPPO/VIVO，:43-53），`POST user/device_token`（`PushService.java`；另有 `DELETE user/device_token` 注销、`POST user/device_badge` 角标同步）。
5. **角标**：`EndpointManager.invoke("push_update_device_badge", num)`（`WKUIKitApplication.stopConn` 时携带未读总数，`WKUIKitApplication.java:192`）→ `registerBadge` + 厂商桌面角标 `OsUtils.setBadge`（`WKPushApplication.java:185-190`）。
6. **厂商配置位置**：华为 appid 在 `wkpush/src/main/AndroidManifest.xml` meta-data `com.huawei.hms.client.appid=108677047`；小米/华为/OPPO key 在 `PushKeys.java:5-9`；vivo `api_key`/`app_id` 在 manifest meta-data；FCM 在 `app/google-services.json`（构建脚本还会按 applicationId 自动改写其 `package_name`，`app/build.gradle:25-37`）。

---

## 5. 对外接口

### 5.1 REST 端点清单（baseUrl = `{api_base_url}/v1/`）

所有接口经 Retrofit + RxJava3 + FastJSON，统一注入 `token` 等公共 header（§4.b）。共 14 个服务接口、约 80 个端点。按模块分组（方法 路径 — 用途）：

**LoginService（wklogin，`LoginService.java`，14 个）**

| 方法 | 路径 | 用途 |
|---|---|---|
| POST | `user/login` | 账号密码登录（明文密码） |
| GET | `user/grant_login` | PC/Web 扫码登录确认 |
| GET | `common/countries` | 国家区号列表 |
| POST | `user/sms/registercode` | 注册短信验证码 |
| POST | `user/sms/forgetpwd` | 找回密码验证码 |
| POST | `user/pwdforget` | 重置密码 |
| POST | `user/register` | 注册 |
| PUT | `user/current` | 更新当前用户资料 |
| POST | `user/sms/login_check_phone` | 登录二次验证发短信 |
| POST | `user/login/check_phone` | 校验验证码并登录 |
| POST | `user/pc/quit` | PC 端下线 |
| PUT | `user/my/setting` | 用户设置 |
| GET | `user/thirdlogin/authcode` | 第三方登录授权码 |
| GET | `user/thirdlogin/authstatus` | 轮询第三方授权状态 |

**MsgService（wkuikit，`MsgService.java`，20 个）**

| 方法 | 路径 | 用途 |
|---|---|---|
| POST | `message/revoke` | 撤回消息 |
| PUT | `coversation/clearUnread` | 清除未读（注意后端拼写 "coversation"） |
| PUT | `message/voicereaded` | 语音已读 |
| GET | `users/{uid}/im` | **获取 IM 长连 tcp_addr（ip:port）** |
| DELETE | `message` | 删除消息 |
| POST | `message/syncack/{last_message_seq}` | 消息同步确认 |
| POST | `message/sync` | 离线消息同步 |
| POST | `message/typing` | 输入中状态 |
| POST | `conversation/sync` | 最近会话同步 |
| POST | `message/channel/sync` | 频道内消息同步 |
| POST | `message/extra/sync` | 消息扩展（点赞等）同步 |
| POST | `message/offset` | 消息偏移 |
| GET | `message/sync/sensitivewords` | 敏感词同步 |
| POST | `conversation/syncack` | 会话同步确认 |
| POST | `message/edit` | 消息编辑 |
| POST | `message/reminder/sync` | 提醒同步 |
| POST | `message/reminder/done` | 提醒完成 |
| POST | `conversations/{channel_id}/{channel_type}/extra` | 会话扩展更新 |
| POST | `conversation/extra/sync` | 会话扩展同步 |
| GET | `message/prohibit_words/sync` | 违禁词同步 |

**UserService（wkuikit，`UserService.java`，15 个）**：`GET users/{uid}`（资料）、`PUT user/current`、`PUT friend/remark`（备注）、`DELETE friends/{uid}`、`POST/DELETE user/blacklist/{uid}`（黑名单）、`GET/POST user/online`（在线状态）、`PUT user/my/setting`、`GET user/qrcode`、`POST/GET user/maillist`（手机通讯录上传/匹配）、`POST user/signal/keys`（Signal 密钥上传）、`POST user/quit`（注销账号）、`GET user/devices/{device_id}`（设备管理）。

**GroupService（wkuikit，`GroupService.java`，14 个）**：`POST group/create`、`POST groups/{groupNo}/members`（加人）、`POST groups/{groupNo}/member/invite`、`GET groups/{groupNo}/member/h5confirm`、`GET groups/{groupNo}`、`GET groups/{groupNo}/membersync`、`PUT groups/{groupNo}/setting`、`PUT groups/{groupNo}`、`DELETE groups/{groupNo}/members`、`PUT groups/{groupNo}/members/{uid}`、`GET groups/{groupNo}/qrcode`、`GET group/my`、`POST groups/{group_no}/exit`、`GET groups/{group_no}/members`。

**WKCommonService（wkbase，`WKCommonService.java`，5 个）**：`GET common/appversion/android/{version}`（升级检查）、`GET common/appconfig`（App 开关，含 `can_modify_api_url`）、`GET channel/state`、`GET channels/{channelID}/{channelType}`、`GET common/appmodule`（动态模块）。

**其余**：FriendService（4）：`POST friend/apply`、`POST friend/sure`、`GET friend/sync`、`PUT users/{uid}/setting`；PushService（3）：`POST/DELETE user/device_token`、`POST user/device_badge`；WKRobotService（2）：`POST robot/sync`、`POST robot/inline_query`；SearchService（1）：`GET user/search`；WKCryptoService（1）：`POST user/signal/getkey`（Signal 公钥获取——**端到端加密基础设施存在但 WKIMUtils.java:131-146 的密钥监听被整段注释，E2EE 疑似停用，待验证**）；ApiService/UploadService（wkbase）：`@GET @Url` 动态 URL 获取上传凭证；IAppService（2）：开放平台 app 信息与授权码（`WKWebViewActivity` 的 `auth` JS handler 使用）。

### 5.2 WKIM SDK 长连接配置

- 初始化：`WKIM.getInstance().init(context, uid, imToken)`（`WKUIKitApplication.java:179`）；文件缓存目录 `wkIMFile`（:175）；debug 模式跟随 `WKBinder.isDebug`（:173，`WKBinder.java:7` = `BuildConfig.DEBUG`）。
- host 来源：`GET /v1/users/{uid}/im` 返回 `tcp_addr`，经 `WKIMUtils.java:148` 的 `addOnGetIpAndPortListener` 喂给 SDK；**SDK 每次建连都回调索取**，支持后端动态调度接入点。
- 连接生命周期：`startChat()`/`stopConn()`（§4.c）、`AliveJobService` 保活、登出 `disconnect(true)`（`WKUIKitApplication.java:555`）。
- 本地消息库：wkbase 自封装 SQLite（`wkbase/db/DBHelper.java`，库名 `{uid}.db`，`DBHelper.java:30`）；WKIM SDK 另有内部消息库（黑盒）。

---

## 6. 部署与运维

### 6.1 构建配置与 OEM 打包

**构建变体一览**：

| 配置项 | debug | release | 出处 |
|---|---|---|---|
| minifyEnabled（app） | false | true | `app/build.gradle` buildTypes |
| shrinkResources | false | true | 同上 |
| jniDebuggable | false | **true（应移除）** | 同上 release 块 |
| signingConfig | 未配置（注释掉） | 未配置（注释掉） | `app/build.gradle:29-46` |
| library 模块混淆 | 均 false | 均 false | 各模块 build.gradle |
| WKBinder.isDebug（HTTP 全量日志开关） | true | false | `WKBinder.java:7`（= BuildConfig.DEBUG） |
| WebView 远程调试 | **开启** | **开启（BridgeWebView 无条件）** | `BridgeWebView.java:69-71` |

**OEM 打包清单（从零打一个自有品牌包需要的全部改动）**：

1. 根目录 `local.properties` 增加 `applicationId=你的包名`（`app/build.gradle:6-23` 强制）；
2. 同步修改 `TSApplication.kt:38`、`MainActivity.kt:20-21` 的 `com.test.ts.R`/`databinding` import（报错文案 `:22` 明示）；
3. 替换 `app/google-services.json`（FCM）；注意构建脚本会自动改写其中 `package_name`（`app/build.gradle:25-37`）；
4. 替换 `wkpush/PushKeys.java` 四组厂商 key、`wkpush/src/main/AndroidManifest.xml` 华为 appid 与 vivo api_key/app_id；按 README 放置华为 `agconnect-services.json`（仓库未含，待验证）；
5. 配置签名（当前 release 无签名配置，§6.1）；
6. 服务端部署 WuKongIM + 唐僧叨叨服务端，客户端首启后在登录页配置 `api_base_url`（或由 MDM/预埋 SP 写入，否则崩溃，`TSApplication.kt:114`）；
7. 替换应用图标/名称（`app/src/main/res/mipmap-*`、`strings.xml`）。


- **applicationId 注入（OEM 核心机制）**：`app/build.gradle:6-27` 从根目录 `local.properties` 读 `applicationId`，同时用作 `namespace` 与 `applicationId`；缺失则 `throw new GradleException`（:21-23）。构建脚本还**自动改写 `app/google-services.json` 的 `package_name`** 为该 applicationId（:25-37）——FCM 配置被静默修改，OEM 若忘记整体替换 google-services.json，会拿着唐僧叨叨官方 Firebase 工程（`tsdddemo`）+ 自己包名的组合，推送必然异常。
- **服务器地址不进包**：见 §4.a，`api_base_url` 运行时配置，未配置即启动崩溃（`TSApplication.kt:114`）；后端开关 `common/appconfig.can_modify_api_url` 控制登录页是否暴露修改入口（`WKLoginActivity.java:191-201`）。
- **源码残留需手工改**：app 壳包名 `com.test.demo2`、资源 import `com.test.ts.R`/`com.test.ts.databinding`（`TSApplication.kt:38`、`MainActivity.kt:20-21`），OEM 换包名要同步改 import。
- **签名**：`app/build.gradle:29-46` 的 debug/release signingConfigs **整段被注释**（原配置 `demo.jks` / 密码 `demo1365` 在注释中可见），release 构建默认不签名，需在 IDE/CI 另行配置；注释中的 keystore 密码属轻度信息泄露。
- **release 构建**：`minifyEnabled true` + `shrinkResources true` + 默认 ProGuard + `app/proguard-rules.pro`（keep AppCompatActivity/Fragment/databinding/自定义 View 等，规则粗糙）；**release 块中 `jniDebuggable true`**——发布包允许 native 调试，应移除。library 模块均 `minifyEnabled false`（混淆集中在 app 壳）。
- **ABI**：app `abiFilters "armeabi-v7a", "arm64-v8a"`；`wkuikit` 声明含 x86/x86_64 的 abiFilters（对 library 无实际作用，属无效配置）。
- **版本管理**：`versionCode 1` / `versionName "1.0"` 硬编码（app/build.gradle defaultConfig），无 CI 注入；升级检查走 `GET common/appversion/android/{version}`，更新弹窗确认后下载 APK（`WKDialogUtils.java:415-418` → `DownloadApkUtils.downloadAPK`，下载到公共 Downloads 目录，`DownloadApkUtils.java:59-68`）。
- MultiDex 开启；`android.injected.testOnly=false`（gradle.properties）。

### 6.2 多渠道/厂商推送配置

OEM 需替换（见 §4.d）：`PushKeys.java`（小米/华为/OPPO）、`wkpush/src/main/AndroidManifest.xml`（华为 appid、vivo api_key/app_id）、`app/google-services.json`（FCM）、华为 `agconnect-services.json`（README 提及，仓库 app 目录未见该文件，待验证是否需 OEM 自行放置）。

版本陈旧度：小米 `MiPush_SDK_Client_3_7_5.jar`（约 2019 年）、OPPO `push-3.0.0.aar`、vivo `v3.0.0.4_484`（约 2020 年）、华为 HMS push 6.1.0.300（2021 年）、FCM BOM 32.8.1（2024 年）。国内四通道 SDK 远落后于厂商当前版本，存在通道兼容性与厂商审核风险。

### 6.3 权限与组件暴露

`app/src/main/AndroidManifest.xml` 声明的敏感权限：

| 权限 | 行 | 备注 |
|---|---|---|
| `MANAGE_EXTERNAL_STORAGE` | :43 | 全量外存，Play 上架需专项申报，超出 IM 最小权限 |
| `READ_LOGS` | :71 | 系统签名级权限，普通环境不授予（声明无效但暴露意图） |
| `SYSTEM_ALERT_WINDOW` / `SYSTEM_OVERLAY_WINDOW` | :56-57 | 悬浮窗（RTC 通话悬浮？待验证） |
| `REQUEST_INSTALL_PACKAGES` | :82 | 配合应用内 APK 升级 |
| `READ_CONTACTS` / `CALL_PHONE` / `READ_PHONE_STATE` | :58,60,18 | 通讯录匹配/直拨/设备信息 |
| 定位/蓝牙/WiFi 状态 | :22-35 | 高德定位注释残留（仓库未见高德 SDK 依赖，待验证） |
| `MOUNT_UNMOUNT_FILESYSTEMS` / `WRITE_MEDIA_STORAGE` | :12-17 | 保护级权限，无效声明 |

组件暴露：主 Activity `exported="true"` 仅含 LAUNCHER intent-filter，正常；FileProvider 非导出（:130-138）。`wkpush` manifest 中小米 `NetworkStatusReceiver`、`PingReceiver` 及 vivo `CommandClientService exported="true"`（带厂商权限保护）按厂商 SDK 要求声明。

---

## 7. 安全审计

标注约定：**[CONFIRMED]** = 代码中已核实（附 file:line）；**[PLAUSIBLE]** = 基于代码结构推断、需动态验证。仅静态审查，未做动态利用验证。

### 7.1 网络层证书校验全面失效 — [CONFIRMED]（高危）

1. **自定义 X509TrustManager 空实现，信任所有证书**：
   - `wkbase/src/main/java/com/chat/base/net/OkHttpUtils.java:51-66`：`checkClientTrusted`/`checkServerTrusted` 方法体为空，`getAcceptedIssuers` 返回空数组；
   - `wkbase/src/main/java/com/chat/base/net/SSLSocketClient.java:25-38`：同样的 trust-all TrustManager；`SSLContext.getInstance("SSL")`（:15，未指定 TLS 版本）。
2. **HostnameVerifier 恒真**：`SSLSocketClient.java:42-44` — `(s, sslSession) -> true`，任何主机名都通过。
3. 该 OkHttpClient 用于**全部 REST 流量**（`RetrofitUtils.java:31`），Glide 图片加载也复用 OkHttp（`WKBaseApplication.java:84` 替换 `OkHttpUrlLoader.Factory`）。
4. **WebView 忽略所有 SSL 错误**：`wkbase/src/main/java/com/chat/base/jsbrigde/BridgeWebViewClient.java:28-30` — `onReceivedSslError` 中 `sslErrorHandler.proceed()`。
5. **明文流量放行**：`app/src/main/AndroidManifest.xml:116` `android:usesCleartextTraffic="true"`；登录页设置服务器地址时**默认补 `http://`**（`WKLoginActivity.java:176`、`ThirdLoginActivity.kt:66`）。

**综合影响**：在「HTTP 服务器地址 + 信任所有证书 + 不校验主机名」的组合下，中间人可完整接管 REST API（窃取/伪造 token、消息、通讯录），并篡改 `GET users/{uid}/im` 响应把长连接指向攻击者服务器，进而代理全部 IM 流量。这是本仓库最严重的安全问题。

### 7.2 token 与敏感数据明文存储 — [CONFIRMED]（高危）

- `WKConfig.java:34-48`：token / im_token 分别存于 SP 键 `wk_token`、`wk_im_token`；uid 存 `wk_uid`（:26-32）；整个 `UserInfoEntity`（含 token 字段）Gson 序列化存 `user_info`（:85-88）。
- 底层为普通 `SharedPreferences`（文件名 `wkSharedPreferences`，MODE_PRIVATE，`WKSharedPreferencesUtil.java:22-23`），**未使用 EncryptedSharedPreferences / Keystore**。
- `app/src/main/AndroidManifest.xml:106` `android:allowBackup="true"`：Android 11 及以下可经 `adb backup` 导出 SP；仓库虽提供 `backup_rules.xml`/`data_extraction_rules.xml`（`app/src/main/res/xml/`），但 `allowBackup=true` 本身仍扩大提取面（规则文件具体内容待验证）。
- 本地消息 SQLite（`{uid}.db`，`DBHelper.java:30`）未见 SQLCipher 等加密。

**本地敏感数据存储清单**（均明文）：

| 数据 | 位置 | 写入点 |
|---|---|---|
| API token | SP `wk_token` | `WKConfig.java:34-36` |
| IM token | SP `wk_im_token` | `WKConfig.java:42-44` |
| uid / 用户名 | SP `wk_uid` / `wk_name` | `WKConfig.java:26-32,50-52` |
| 完整 UserInfoEntity（含 token 字段） | SP `user_info`（Gson JSON） | `WKConfig.java:85-88` |
| App 配置 | SP `app_config` | `WKConfig.java:68-71` |
| 服务器地址 | SP `api_base_url` | `TSApplication.kt:110` 读取 |
| 敏感词库 | SP `wk_sensitive_words` | `WKUIKitApplication.java:156` 读取 |
| 设备 ID（ANDROID_ID 或随机 UUID） | SP `device_id` | `WKConstants.java:48-61` |
| 锁屏时间戳 | SP `lock_start_time` | `TSApplication.kt:163-164` |
| 消息/联系人库 | SQLite `{uid}.db`（wkbase 自封装） | `DBHelper.java:30` |
| 崩溃日志（含设备信息+堆栈） | 存储目录 `wkCrash/*.log` | `CrashHandler.java:160-196` |

- token 无本地过期/刷新机制代码（后端策略待验证）。

### 7.3 密码明文传输（客户端未哈希） — [CONFIRMED]（中危）

- `wklogin/src/main/java/com/chat/login/service/LoginModel.java:53-55`：`jsonObject.put("password", pwd)`；登录/注册（`registerApp`）/重置密码同样不哈希。安全性完全依赖 TLS——而 TLS 已被 §7.1 架空。

### 7.4 日志泄露敏感信息 — [CONFIRMED]（中危）

- `wkbase/src/main/java/com/chat/base/act/WKWebViewActivity.java:152-154`：访问 `report.html` 时把 **uid + token 拼进 URL query**（:152），随后 `Log.e("加载的URL", url)`（:154）将**含 token 的完整 URL 打入 logcat**。
- `wkpush/src/main/java/com/chat/push/WKPushApplication.java:96,114,153,205`：华为/OPPO/vivo/FCM 推送 token 以 `Log.e` 明文输出；`wkpush/.../push/WKFirebaseMessagingService.java:18` 同样输出 FCM token。这些 Log **无任何 debug 开关**，release 包同样输出。
- `wkbase/src/main/java/com/chat/base/net/LogInterceptor.java:38-105`：debug 构建下记录全部请求 header（含 `token`）与完整响应体；开关为 `WKBinder.isDebug = BuildConfig.DEBUG`（`WKBinder.java:7`），release 下关闭——风险限于 debug 包。
- 其他零散输出：`WKUIKitApplication.java:186` `Log.e("去连接", ...)`、`AliveJobService.java:47`、`WKScanApplication.java` 二维码识别内容等。
- `CrashHandler`（`wkbase/.../utils/CrashHandler.java:160-196`）将崩溃堆栈写入存储目录 `wkCrash`（经 `WKFileUtils.getNormalFileSavePath`，外置应用目录），并 `EndpointManager.invoke("saveCrashFile", ex)`（:121）外挂上报；`wkuikit/setting/ErrorLogsActivity.kt` 提供错误日志查看页——崩溃日志可能含上下文敏感数据。

### 7.5 WebView / JSBridge 攻击面 — [CONFIRMED + PLAUSIBLE]（高-中危）

`WKWebViewActivity`（`wkbase/src/main/java/com/chat/base/act/WKWebViewActivity.java`）：

- `:145-148` 从 intent extra `url` 取地址，**不校验域名白名单**，无 scheme 时补 `http://` 直接 `loadUrl`（:155）。该 Activity 被多处以显式 intent 唤起（扫码结果回调 `WKScanApplication.java` 的 `showWebView`、文本消息链接、关于页 `WKAboutActivity.java:39` 等）——**扫一个二维码即可让 App 内嵌浏览器加载任意 http(s) 页面**（[CONFIRMED] 机制；实际危害取决于入口可达性，[PLAUSIBLE]）。
- `:161` `setJavaScriptEnabled(true)`；`:178` `setMixedContentMode(0)`（MIXED_CONTENT_ALWAYS_ALLOW，https 页面可加载 http 资源）；`:172` 开启 DOM storage；`:163` `setPluginState(ON)`（过时 API）。
- **JS 桥**：`registerHandler("quit"/"auth"/"getChannel"/"showConversation")`（:219-255）——网页可关闭页面、触发第三方应用授权弹窗（用户确认后把授权 code 回调给网页，:375-436）、**读取当前会话 channelID/channelType（`getChannel`，:237-242）**、跳转指定会话。**任何被加载的页面都可调用这些 handler**——恶意页面可探测用户正在哪个频道聊天，或诱导用户完成授权（[CONFIRMED] 机制存在；实际利用依赖诱导用户加载恶意页面，[PLAUSIBLE]）。
- 桥接实现走 URL scheme 拦截（`BridgeWebViewClient.shouldOverrideUrlLoading`，:33-89），**未使用 `addJavascriptInterface`**（全仓库 grep 无结果），规避了经典 addJavascriptInterface RCE，值得肯定。
- `wkbase/src/main/java/com/chat/base/jsbrigde/`（8 个文件）是知名开源 JSBridge 库（lzyzsd/JsBridge 风格，`BridgeWebView`/`BridgeWebViewClient`/`WebViewJavascriptBridge`，与唐僧叨叨生态的独立 JSBridge 仓库同源）的 **vendored 拷贝**：本仓库内嵌其 Android 端实现作为 `WKWebViewActivity` 的桥。
- `BridgeWebView.java:69-71`：**无条件** `WebView.setWebContentsDebuggingEnabled(true)`（不区分 build type）——release 包 WebView 可被 chrome://inspect 远程调试，结合敏感页面可读取 DOM 中 token 等数据。[CONFIRMED]
- WebView SSL 放行（§7.1 第 4 点）叠加任意 URL 加载，WebView 内 https 亦无保障。

### 7.6 FastJSON 反序列化面 — [CONFIRMED 版本陈旧 / PLAUSIBLE 利用]（中危）

- `wkbase/build.gradle`：`com.alibaba:fastjson:1.2.83`（fastjson 1.x 最终版，2021 年，已停更）。Retrofit 全量响应经 `FastJsonResponseBodyConverter.kt:13` `JSON.parseObject(String, Type)` 反序列化；`WKUIKitApplication.java:158` 等处直接 `JSON.parseObject(json, SensitiveWords.class)`。
- 按 Type 解析时 autotype 默认关闭，直接 RCE 利用面小；但 1.2.83 存在公开 DoS（深层嵌套 JSON）与历史 autotype 绕过链。服务端响应若被 MITM 替换（§7.1 使其可行），恶意 JSON 直达解析器 [PLAUSIBLE]。建议迁移 `com.alibaba.fastjson2:fastjson2`。

### 7.7 硬编码密钥 / 测试配置 — [CONFIRMED]（中危）

- `wkpush/src/main/java/com/chat/push/PushKeys.java:5-9`：小米 `xiaoMiAppID=2882303761520252187` / `xiaoMiAppKey=5302025297187`、华为 `huaweiAPPID=108677047`、**OPPO `oppoAppKey`/`oppoAppSecret` 明文**（AppSecret 属服务端级凭证，泄露可被冒用推送）。
- `wkpush/src/main/AndroidManifest.xml`：vivo `com.vivo.push.api_key=d14dac0803bc8470d9ace9e29657a160`、`com.vivo.push.app_id=105660344`（meta-data 段）。
- `app/google-services.json`：真实 Firebase 工程 `tsdddemo`（project_number 232134567764）及 API key `AIzaSyDOVB7GuZnDXIZWT-qaIiCrQU4qvfwWKGU`（FCM key 需配合包名+签名，泄露风险中低，但暴露官方测试工程；`mobilesdk_app_id` 形如伪造占位 `4454b65123eds3092326735`，含 "eds" 字母，疑似手工篡改过，待验证）。
- `app/build.gradle:29-46`（注释中）：签名文件 `demo.jks` 及密码 `demo1365`。
- 未发现硬编码 IM/API 服务器地址（符合"运行时配置"设计）；`WKBaseApplication.java:52` 硬编码 `appID = "wukongchat"`（随请求 header 上送）。

### 7.8 应用内升级 APK 无完整性校验 — [CONFIRMED 机制 / PLAUSIBLE 利用]（高危）

- 升级流程：`GET common/appversion/android/{version}` 返回 `download_url` → 更新弹窗确认 → `WKDialogUtils.java:415-418` 以 `WKApiConfig.getShowUrl(versionEntity.download_url)` 下载 → `DownloadApkUtils.downloadAPK`（`DownloadApkUtils.java:59-68`）存到公共 Downloads 目录后调起安装（manifest 声明 `REQUEST_INSTALL_PACKAGES`，:82）。
- **全程无签名/哈希校验**，下载地址来自服务器响应且允许 http（§7.1）——MITM 可篡改 `appversion` 响应投递恶意 APK，诱导用户一键安装。属"机制确认存在、利用依赖 MITM"的高危链。

### 7.9 native 库与二进制面 — [PLAUSIBLE]（低-中危）

- `MyLibs/rlottie/lottie_release.aar`：Telegram rlottie 衍生，含 native .so（armeabi-v7a/arm64-v8a），rlottie 历史上有多个 JSON 解析内存破坏 CVE；本应用用它渲染聊天贴纸/Lottie 动画，**处理对端发送的动画 JSON 属远程可达攻击面**。AAR 无版本、无来源记录，是否含补丁未知（待验证）。
- OPPO/vivo AAR、MiPush jar 版本陈旧（§6.2），厂商 SDK 历史漏洞不随仓库更新。
- release 包 `jniDebuggable true`（`app/build.gradle` release 块）放大 native 调试面。

### 7.10 其他观察

- **SQL 字符串拼接**（低危）：`wkuikit/.../db/WKContactsDB.java:103-104` `isExist` 将通讯录 phone/name 直接拼进 SQL（数据来源为本机通讯录，注入面小，但属坏实践；同文件 :33 亦有 rawQuery）。`wkbase/db/WKBaseDBManager.java:51` `execSQL(sql)` 执行传入 SQL，需审计各调用方（待验证）。
- `TSApplication.kt:114`：`api_base_url` 缺失时 Application 直接抛异常崩溃——可用性缺陷，崩溃信息同时泄露内部配置键名。[CONFIRMED，低危]
- `MainActivity.kt:86-89`：**隐私政策点击事件方法体只构造了字符串、没有调用 `showWebView`**（对比 :74-76 用户协议正常）——隐私政策链接点击无效，属合规缺陷。[CONFIRMED]
- `EndpointManager.invoke` 无访问控制：任何模块可调用任何端点，包括 `"uikit_sql"`（`WKUIKitApplication.java:220` 注册的数据库调试菜单，[PLAUSIBLE] 若界面可达则泄露消息库）。
- `WKConstants.java:48-61`：device_id 使用 `ANDROID_ID`，合规上优于 IMEI（manifest 声明 READ_PHONE_STATE 但代码未见 getImei 调用，待验证）。
- `AndroidManifest.xml:43` `MANAGE_EXTERNAL_STORAGE`：全量存储访问，超出 IM 应用最小权限原则，为 Play 上架高敏权限。
- WKIM SDK 的 `setDebug(WKBinder.isDebug)`（`WKUIKitApplication.java:173`）——debug 包 IM SDK 日志粒度未知（黑盒）。

---

## 8. 代码质量评估

### 8.1 规模与 TOP 文件

全仓库（不含 MyLibs 二进制、含测试 stub）约 **9.24 万行** Java/Kotlin。TOP 大文件（`wc -l`）：

| 行数 | 文件 |
|---|---|
| 2710 | `wkuikit/.../chat/ChatPanelManager.kt` |
| 2668 | `wkuikit/.../chat/ChatActivity.java` |
| 2595 | `wkbase/.../views/scroll/ConsecutiveScrollerLayout.java` |
| 2115 | `wkbase/.../views/RecyclerListView.java` |
| 1858 | `wkbase/.../ui/components/BottomSheet.java` |
| 1780 | `wkbase/.../utils/SvgHelper.java` |
| 1621 | `wkbase/.../msgitem/WKChatBaseProvider.kt` |
| 1586 | `wkbase/.../ui/components/AlertDialog.java` |
| 1581 | `wkbase/.../views/swipeback/ViewDragHelper.java` |
| 1546 | `wkuikit/.../chat/provider/SelectTextHelper.kt` |
| 1204 | `wkbase/.../ui/components/NumberPicker.java` |
| 1083 | `wkuikit/.../chat/provider/WKTextProvider.kt` |
| 1082 | `wkuikit/.../fragment/ChatFragment.java` |
| 1052 | `wkbase/.../utils/WKFileUtils.java` |

特征：**UI 上帝类严重**——`ChatActivity` + `ChatPanelManager` 合计约 5400 行承载聊天页全部逻辑；`wkbase/views`、`wkbase/ui/components` 大量文件是从 Telegram for Android 移植的控件（RecyclerListView、BottomSheet、AlertDialog、NumberPicker、ConsecutiveScrollerLayout 的代码风格与包内 `org.telegram.*` 引用一致，`WKBaseApplication.java:34` import `org.telegram.lottiejson.LottieUtils`、`TabActivity.java:59` 使用 `org.telegram.ui.Components.RLottieImageView`），**无移植来源说明与许可证头**——Telegram Android 为 GPLv2，本仓库以 Apache-2.0 发布存在许可证冲突（合规风险，待法律确认）。

### 8.2 Kotlin/Java 混编

| 模块 | Kotlin | Java | 说明 |
|---|---|---|---|
| app | 4 | 1 | 壳为 Kotlin |
| wkbase | 65 | 262 | 老代码 Java，新文件（converter、部分组件）Kotlin |
| wkuikit | 50 | 144 | 同上 |
| wklogin | 4 | 20 | 主体 Java |
| wkscan | 0 | 8 | 纯 Java |
| wkpush | 0 | 13 | 纯 Java |
| 合计 | 123 | 448 | Java 占 78% |

- 混编无规范：同类功能两种语言并存（`WKChatBaseProvider.kt` vs 大量 Java Provider；`FastJsonConverterFactory.kt` vs Java 网络层）。
- Java 代码风格停留在 2019-2020 年：内部类单例、满屏 `getInstance()`、无空安全、`assert` 用于运行期判空（`WKWebViewActivity.java:112,146`，release 中 assert 不生效）。
- 单例泛滥：几乎每个 Manager/Model/Application 都是饿汉/内部类/枚举单例（`WKConfig`、`WKSharedPreferencesUtil` 枚举单例、`EndpointManager`、`LoginModel`、`MsgModel`、`PushModel`……），全局状态互相引用，不可测试。
- 网络层模式统一（`WKBaseModel.request/quest` + RxJava 线程切换，`WKBaseModel.java:17-60`），但错误处理大量空 `onFail`（如 `MsgModel.java:257-259,274-276`），失败被静默吞掉。

### 8.3 测试与工程化

- **测试**：仅 AS 模板 stub（`ExampleUnitTest.kt` 验证 `2+2=4`、`ExampleInstrumentedTest` 验证包名），核心业务零覆盖。字符串路由（EndpointManager）+ 全局单例的架构使单测成本极高，重构无安全网。
- **注释**：中文注释覆盖率尚可（类头多带日期+用途，如 `// 2020-07-17 14:55`），但**大段注释掉的死代码遍布全仓**：`WKPushApplication.java:72-82,221-244`（被注释的通道初始化与 FCM 备用逻辑）、`app/build.gradle:29-46`（签名段）、`WKIMUtils.java:131-146`（Signal 密钥监听整段注释，E2EE 疑似停用）、`TabActivity.java:135`、`RetrofitUtils.java:35` 等。信号噪声比差，且注释代码暴露了演进中的敏感信息（keystore 密码）。
- **残留包名/标记**：
  - app 壳源码包名 `com.test.demo2`（测试占位包名进入生产代码）；
  - 资源引用 `com.test.ts.R` / `com.test.ts.databinding`（`TSApplication.kt:38`、`MainActivity.kt:20-21`）；
  - 测试目录包名 `com.xinbida.tsdd.demo`（app）、`com.chat.im.limlogin`（wklogin）等历史痕迹；
  - 布局文件名拼写 `act_webvieiw_layout`（webvieiw，`WKWebViewActivity.java:78`）、`wkuikit/enity/`（entity 拼写）、后端字段 `coversation/clearUnread`（`MsgService.java:35`）、`wkbase/jsbrigde/`（jsbridge 拼写）。
- README 信息过期（徽标 compileSdk 33/minSdk 23 vs 实际 35/24）。

### 8.4 架构层面

- **优点**：模块边界清晰、单向依赖；EndpointManager 让功能模块可插拔（README 的"二次开发"承诺基本兑现）；消息渲染 Provider 体系扩展性好；同步协议（消息/会话/extra/reminder 分通道 sync + syncack）设计完整；推送五通道抽象统一（`PushModel.registerDeviceToken` 单点上报）。
- **缺点**：
  - EndpointManager 字符串契约无编译期保障，存在死调用（`update_base_url`，§3.2）与无主的隐式端点（`rtc_is_calling`、`chow_check_lock_screen_pwd` 依赖外挂模块，缺席时静默返回 null）；
  - Retrofit 单例缓存 baseUrl（`RetrofitUtils.java:27-34`），改服务器地址不生效需重启；
  - 静态可变全局：`WKApiConfig.baseUrl/baseWebUrl` 为 public static（`WKApiConfig.java:12-13`），任意代码可改写；
  - `WKUIKitApplication`（760 行）等"模块 Application"聚合过多职责（初始化+消息注册+菜单注册+通知逻辑）；
  - `invoke` 线性扫描 + `ConcurrentHashMap` 混用 `ArrayList`（`EndpointManager.java:25-37`），列表并发修改与排序（`:101`）存在竞态窗口（低概率，待验证）。

---

## 9. 风险与改进建议

### 9.1 风险汇总表

| # | 风险 | 等级 | 类型 | 出处 |
|---|---|---|---|---|
| R1 | 全量 REST/Glide 流量信任所有证书 + 不校验主机名 | 高 | 安全 [CONFIRMED] | `OkHttpUtils.java:51-66`、`SSLSocketClient.java:25-44` |
| R2 | WebView `onReceivedSslError` 一律 proceed + 明文流量放行 + 服务器地址可配 http | 高 | 安全 [CONFIRMED] | `BridgeWebViewClient.java:28-30`、Manifest:116、`WKLoginActivity.java:176` |
| R3 | token/im_token/userInfo 明文 SharedPreferences + allowBackup=true + 消息库未加密 | 高 | 安全 [CONFIRMED] | `WKConfig.java:34-48,85-88`、Manifest:106、`DBHelper.java:30` |
| R4 | WebView 任意 URL 加载（扫码/链接直达）、JS 桥暴露频道信息、WebView 调试无条件开启、token 拼 URL 并打日志 | 高 | 安全 [CONFIRMED] | `WKWebViewActivity.java:145-161,152-154,219-255`、`BridgeWebView.java:69-71` |
| R5 | 应用内升级 APK 无签名校验，下载地址可被 MITM 篡改投递恶意包 | 高 | 安全 [CONFIRMED 机制] | `WKDialogUtils.java:415-418`、`DownloadApkUtils.java:59-68`、Manifest:82 |
| R6 | 厂商推送密钥硬编码（含 OPPO AppSecret、vivo api_key、FCM 配置入库） | 中 | 安全 [CONFIRMED] | `PushKeys.java:5-9`、wkpush Manifest、`google-services.json` |
| R7 | 推送 token / 含 token URL 无条件 Log.e 输出 | 中 | 安全 [CONFIRMED] | `WKPushApplication.java:96,114,153,205`、`WKWebViewActivity.java:154`、`WKFirebaseMessagingService.java:18` |
| R8 | FastJSON 1.2.83 停更版本承担全部响应反序列化 | 中 | 安全/供应链 [CONFIRMED 版本] | `wkbase/build.gradle`、`FastJsonResponseBodyConverter.kt:13` |
| R9 | JitPack 依赖 + 无校验的本地 AAR/JAR（rlottie native、厂商推送 SDK 陈旧） | 中 | 供应链 | `settings.gradle`、`MyLibs/`、`wkpush/libs/` |
| R10 | 登录密码客户端不哈希（配合 R1 等于明文可截） | 中 | 安全 [CONFIRMED] | `LoginModel.java:53-55` |
| R11 | 隐私政策链接点击无效；首启未配 api_base_url 即崩溃 | 中 | 合规/可用性 [CONFIRMED] | `MainActivity.kt:86-89`、`TSApplication.kt:114` |
| R12 | Telegram(GPLv2) 代码移植无来源声明，与 Apache-2.0 发布存在许可证冲突 | 中 | 合规 [PLAUSIBLE] | `wkbase/views/*`、`org.telegram.*` 引用 |
| R13 | EndpointManager 字符串路由死调用（`update_base_url` 无注册方）、改地址不重建 Retrofit | 中 | 质量 [CONFIRMED 死调用] | `WKLoginActivity.java:180`、`RetrofitUtils.java:27-42` |
| R14 | release 无签名配置（被注释，密码入库）、release `jniDebuggable true` | 中 | 构建安全 [CONFIRMED] | `app/build.gradle:29-46`、release 块 |
| R15 | 零测试 + 上帝类（ChatActivity 2668 行等）+ 全局单例，重构无安全网 | 中 | 质量 | §8 |
| R16 | 权限过度（MANAGE_EXTERNAL_STORAGE、READ_LOGS、SYSTEM_ALERT_WINDOW） | 低-中 | 合规 [CONFIRMED] | `app/src/main/AndroidManifest.xml:43,56,71` |
| R17 | E2EE（Signal）基础设施存在但密钥监听被注释停用，"加密 IM"名实不符风险 | 低-中 | 安全 [CONFIRMED 注释 / 待验证] | `WKIMUtils.java:131-146`、`UserService.java`、`WKCryptoService.java` |
| R18 | SQL 字符串拼接（通讯录查询）、CrashHandler 崩溃日志落外置目录 | 低 | 安全 [CONFIRMED] | `WKContactsDB.java:103-104`、`CrashHandler.java:160-196` |

### 9.2 改进建议（按优先级）

**P0（安全止血）**

1. 删除 trust-all TrustManager 与恒真 HostnameVerifier，恢复系统默认校验；如确有自签名内网部署需求，改用 `network_security_config` 按域钉证书，并关闭 `usesCleartextTraffic`、登录页强制 https 前缀。
2. `BridgeWebViewClient.onReceivedSslError` 改为 `cancel()`；`WebView.setWebContentsDebuggingEnabled` 仅在 `BuildConfig.DEBUG` 开启；`WKWebViewActivity` 增加 URL 白名单（仅 `WKApiConfig.baseWebUrl` 域及显式受信域）并禁止混合内容。
3. token 迁移至 `EncryptedSharedPreferences`（Keystore 主密钥）；删除所有输出 token/推送 token 的 `Log.e`；`report.html` 改用一次性短期票据而非 URL 携带长期 token；`allowBackup="false"` 或在 backup rules 显式排除 `wkSharedPreferences`。
4. 应用内升级增加 APK 签名一致性校验（与当前应用签名比对）+ 强制 https 下载地址。

**P1（供应链与凭证）**

5. fastjson 1.2.83 → fastjson2（或统一到 Gson/Moshi，消除双序列化栈）；为 JitPack/本地 AAR 建立 SBOM 与版本台账，启用 Gradle dependency verification；rlottie 升级至含补丁版本或改用官方 lottie-android。
6. 所有推送 key 移出仓库（local.properties/CI secret 注入），轮换已入库的 OPPO AppSecret、vivo api_key、Firebase 配置；小米/OPPO/vivo SDK 升级到厂商当前版本。
7. 恢复并外置签名配置（CI 注入 keystore），release 去掉 `jniDebuggable true`，版本号 CI 注入；删除注释中的 keystore 密码。

**P2（架构与质量）**

8. EndpointManager：为全部 sid 建常量注册表 + 注册/调用配对 lint 检查；补 `update_base_url` 处理器（调 `RetrofitUtils.resetRetrofit()` 并重建 `WKApiConfig`）或删除死调用；`invoke` 改 sid 直索引 Map。
9. 拆分 `ChatActivity`/`ChatPanelManager`（按面板、消息列表、输入区分层）；核心 Model（LoginModel/MsgModel/PushModel）补单测，从 EndpointManager、WKApiConfig 等纯逻辑入手。
10. 修复隐私政策点击（`MainActivity.kt:87` 补 `showWebView(...)`）；首启未配置 `api_base_url` 时引导进入设置页而非崩溃。
11. 澄清 Telegram 移植代码许可证问题（替换为 Apache 兼容实现或按 GPL 合规处理）；明确 E2EE 策略（恢复 Signal 密钥监听或在宣传中去除加密承诺）；README 徽标与实际 SDK 版本对齐。
12. 权限最小化：评估移除 `MANAGE_EXTERNAL_STORAGE`（改用 SAF/PhotoPicker）、`READ_LOGS`、`SYSTEM_ALERT_WINDOW`；清理高德定位等无效权限声明。

---

## 附：关键文件索引

| 主题 | 文件 |
|---|---|
| 应用入口 | `app/src/main/java/com/test/demo2/TSApplication.kt`、`MainActivity.kt`、`AppFrontBackHelper.java` |
| 网络层 | `wkbase/src/main/java/com/chat/base/net/`（OkHttpUtils、RetrofitUtils、SSLSocketClient、LogInterceptor、FastJson*Converter、CommonRequestParamInterceptor、ApiService） |
| 配置与存储 | `wkbase/src/main/java/com/chat/base/config/`（WKApiConfig、WKConfig、WKConstants、WKSharedPreferencesUtil、WKBinder） |
| 模块通信 | `wkbase/src/main/java/com/chat/base/endpoint/EndpointManager.java`、`EndpointCategory.java`、`EndpointSID.java` |
| WebView/JSBridge | `wkbase/src/main/java/com/chat/base/act/WKWebViewActivity.java`、`wkbase/src/main/java/com/chat/base/jsbrigde/`（8 文件，vendored JsBridge） |
| 登录 | `wklogin/src/main/java/com/chat/login/service/LoginModel.java`、`LoginService.java`、`ui/WKLoginActivity.java`、`ui/ThirdLoginActivity.kt` |
| IM 桥接 | `wkuikit/src/main/java/com/chat/uikit/WKUIKitApplication.java`、`chat/manager/WKIMUtils.java`、`chat/manager/WKSendMsgUtils.java`、`message/MsgModel.java`、`message/MsgService.java` |
| 消息类型 | `wkbase/src/main/java/com/chat/base/msgitem/WKContentType.java`、`WKMsgItemViewManager.java`、`WKChatBaseProvider.kt` |
| 推送 | `wkpush/src/main/java/com/chat/push/WKPushApplication.java`、`OsUtils.java`、`PushKeys.java`、`service/PushModel.java`、`push/`（5 通道回调） |
| 升级/崩溃 | `wkbase/src/main/java/com/chat/base/utils/DownloadApkUtils.java`、`CrashHandler.java`、`WKDialogUtils.java` |
| 构建 | 根 `build.gradle`、`settings.gradle`、`app/build.gradle`、各模块 `build.gradle`、`app/proguard-rules.pro`、`gradle.properties` |

> 不确定项已在正文以"待验证"标注：WKMsgContentType 各类型数值（SDK 黑盒）、`update_base_url` 注册方（疑缺失）、rlottie AAR 补丁状态、backup_rules 具体内容、IM SDK 连接层加密细节、E2EE 是否整体停用、`agconnect-services.json` 归属、google-services.json 中 mobilesdk_app_id 是否被手工篡改。
