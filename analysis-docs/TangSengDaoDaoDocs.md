# TangSengDaoDaoDocs 仓库分析报告

> 分析对象：`/data/code/tsdd-analysis/TangSengDaoDaoDocs`
> 仓库性质：唐僧叨叨（TangSengDaoDao）官方文档站源码（VitePress 静态站点）
> 分析日期：2026-10-12；仓库最近一次 git 提交：`2026-01-01 11:01:25 +0800`（`git log -1`，提交信息 "docs: update"）
> 定位说明：本仓库是唐僧叨叨生态的"官方口径"文档，本报告以文档体系结构、完整性、准确性线索为分析重点，为其余 9 个仓库（Server / Web / Android / iOS / Manager / Cli / appstore 等）的分析提供对照基准。

---

## 1. 仓库概述

**一句话定位**：唐僧叨叨产品的门户 + 开发者文档站，承载"产品介绍 → 演示下载 → 定价售卖 → 部署运维 → 集成 API → 各端二次开发"的完整用户旅程。

**在生态中的角色**：

- **产品门户**：首页（`src/index.md:9`）标语"让企业轻松拥有自己的即时通讯"，配合特性矩阵、演示 Demo（`src/guide/demo.md`）、收费模块定价（`src/price/index.md`），承担营销转化职能。
- **运维手册**：`src/dev/backend/` 提供 5+ 种部署方式（Docker Compose / 小皮面板 / 1Panel / 宝塔 / Rainbond，另有 K8s 占位）、端口说明、全量配置参考，是私有化部署客户的主要操作依据。
- **集成契约**：`src/api/` 记录服务端集成 API（注意：实为 WuKongIM 通讯端 5001 端口的 API，详见第 4b 节辨析），`src/dev/backend/fullconfig.md` 是 TangSengDaoDaoServer 配置的权威参考。
- **二开指南**：`src/dev/{android,ios,web,manager}` 覆盖四个端的编译、配置、结构、模块化开发。
- **生态锚点**：`src/guide/fastentry.md` 列出全部 13 个相关仓库（TangSengDaoDao 系 5 个 + WuKongIM 系 8 个）的 GitHub/Gitee 地址，是生态拓扑的官方声明。

文档站线上地址为 `https://tsdaodao.com`（由 `src/guide/fastentry.md:10-13` 中的自引用链接推知）。页脚版权声明主体为"上海信必达网络科技有限公司"（`.vitepress/config.ts:58`），并挂有两个 ICP 备案号（`.vitepress/config.ts:59`）。

---

## 2. 技术栈与依赖

### 2.1 核心技术栈

| 类别 | 技术 | 版本 | 出处 |
|---|---|---|---|
| 文档引擎 | VitePress | **1.0.2**（精确锁定） | `package.json:23` |
| UI 框架 | Vue | ^3.4.21 | `package.json:24` |
| 构建 | Vite | ^5.2.8 | `package.json` devDependencies |
| CSS 引擎 | UnoCSS | ^0.59.0 | `package.json`；`uno.config.ts`（presetUno + presetIcons + presetAttributify + transformerDirectives） |
| 代码块图标 | vitepress-plugin-group-icons | ^1.2.4 | `package.json`；`.vitepress/config.ts:3,31,82-97` |
| 图片灯箱 | markdown-it-custom-attrs（注入 `data-fancybox`）+ Fancybox（本地静态资源） | ^1.0.2 | `package.json`；`.vitepress/config.ts:19-20,30` |
| 图标集 | @iconify-json/{carbon,logos,mdi,ph,simple-icons,twemoji,vscode-icons} | — | `package.json` |
| 语言 | TypeScript（config 层） | ^5.0.4 | `package.json` |

### 2.2 构建与主题定制

- **包管理双轨**：`package.json:45` 声明 `packageManager: pnpm@9.15.4`，仓库内同时存在 `pnpm-lock.yaml` 与 `yarn.lock`；但 `Dockerfile` 第 4-9 行实际使用 **yarn** 安装构建（`curl ... yarnpkg.com/install.sh | bash` → `yarn install` → `yarn build`）。README 与 `scripts`（dev/build/serve）则按 pnpm 描述。**pnpm/yarn 混用是已确认的不一致点**。
- **VitePress 配置要点**（`.vitepress/config.ts`）：
  - `lang: "zh-CN"`（第 14 行），`srcDir: "./src"`（第 35 行），`lastUpdated: true`（第 34 行）；
  - **`ignoreDeadLinks: true`（第 15 行）**——构建期忽略死链，意味着死链不会被 CI/构建拦截，全靠人工发现；
  - 本地搜索（`search.provider: "local"`，第 61-62 行），中文文案全套定制（上次更新/目录/上一篇/下一篇，第 41-46 行）；
  - head 注入：百度统计（第 21 行 `hm.js?945767791d09113d72014cd801857369`）与第三方客服 widget tgo-ai（第 22 行，含明文 api_key，详见第 7 节）。
- **主题扩展**：`.vitepress/theme/index.ts` 继承默认主题，在 `home-features-after` 插槽挂载 `HomePreview.vue`；自定义样式 `styles/var.css`、`styles/custom.css`；`.vitepress/builtin.ts` 提供 group-icons 自定义图标。
- **动态版本号注入**：`.vitepress/utils/fetchReleaseTag.js:2` 硬编码 `releaseTag = "1.1.2"`，首页加载时把该版本号 `<samp>` 追加到 hero tagline（`src/index.md:43-50` 调用）。该值是**文档站自述的产品版本口径**，需与各端仓库实际 release 对账。

---

## 3. 目录结构与内容架构

### 3.1 目录树（裁剪，省略图片资源）

```
TangSengDaoDaoDocs/
├── Dockerfile                  # 两阶段构建：node:18 构建 + nginx 托管
├── Makefile                    # build / deploy（推阿里云镜像仓库）
├── docker-entrypoint.sh        # envsubst 渲染 nginx 模板
├── nginx.conf.template         # SPA fallback 配置
├── package.json / pnpm-lock.yaml / yarn.lock
├── uno.config.ts
├── .vitepress/
│   ├── config.ts               # 站点主配置
│   ├── navbar.ts               # 顶部导航
│   ├── sidebar.ts              # 侧边栏（按路径分区）
│   ├── builtin.ts              # group-icons 自定义图标
│   ├── theme/                  # 主题扩展（HomePreview.vue、样式）
│   └── utils/fetchReleaseTag.js # 首页版本号硬编码 "1.1.2"
└── src/                        # 内容根（srcDir）
    ├── index.md                # 首页（home layout）
    ├── guide/                  # 用户向指南（7 篇 + 下载二维码）
    ├── price/                  # 定价（index + detail）
    ├── api/                    # 集成 API（7 篇：index/message/user/channel/conversation/datasource/webhook）
    ├── dev/
    │   ├── backend/            # 后端部署与二开（20 篇 md）
    │   ├── android/            # Android（8 篇）
    │   ├── ios/                # iOS（9 篇）
    │   ├── web/                # Web/PC（7 篇）
    │   └── manager/            # 后台管理（5 篇 + user-manual/ 19 篇）
    └── public/                 # 静态资源（logo、favicon、视频、fancybox）
```

全仓 markdown 共 **83 篇**（含根 README.md），站点内容 82 篇，总计约 **5312 行**（`wc -l` 实测，详见第 8 节篇幅分布）。

### 3.2 内容地图

```mermaid
mindmap
  root((唐僧叨叨文档站))
    首页/门户
      首页 index
      演示Demo guide/demo
      收费模块 price/index + price/detail
    guide 指南
      什么是唐僧叨叨
      快速入口 fastentry
      问题解决途径 support
      线上Demo体验 demo
      快速搭建 quickstart(侧栏隐藏)
      法律声明 legal
      常见问题 question(占位)
    api 集成API(导航孤岛)
      基础 index(5001端口警告/route)
      消息 message
      用户 user
      频道 channel
      最近会话 conversation
      Datasource
      Webhook
    dev/backend 后端
      部署 compose/xp-panel/1panel/btpanle/rainbond
      一键部署 deploy(CLI,侧栏隐藏)
      k8s(占位) docker-install deploy-mysql
      端口 port 验证 verify 配置 fullconfig
      架构 architecture 流程 workflow
      环境 devenv 结构 structure
      模块引入 moduleimport API(外链apidocs.botgate.cn)
      FAQ question
    dev/android
      intro build config structure module
      aar_import code_import question
    dev/ios
      intro build config structure module
      moduleimport package_notes question
    dev/web
      intro build config structure
      module(待补充) moduleimport question
    dev/manager
      intro build config structure question(空)
      用户手册 user-manual 19篇
        通用/首页/用户/群组/消息/举报/工具/设置
        工作台 workplace(侧栏隐藏)
```

### 3.3 navbar / sidebar 组织逻辑

**顶部导航**（`.vitepress/navbar.ts:3-46`）：指南 → 后端 → iOS → Android → Web → 后台管理 → 收费模块 → 演示Demo → GitHub Star 徽章。注意：

- **导航中没有 "API 文档" 入口**；`src/api/` 整个分区在 navbar 和 sidebar 中均无注册，属于"导航孤岛"（仅可直接 URL 访问，或从 `src/api/*.md` 内部互链）。sidebar 中唯一的"API文档"项是指向外部站点 `https://apidocs.botgate.cn/` 的外链（`.vitepress/sidebar.ts:58`）。
- "后端"导航直接落在 `deploy-compose`（部署页），而非架构/介绍页，体现"部署优先"的产品导向。

**侧边栏**（`.vitepress/sidebar.ts`）按路径前缀分区（`/guide`、`/dev/backend`、`/dev/ios`、`/dev/android`、`/dev/web`、`/dev/manager`），每区固定"快速开始 / 二次开发 / 其他"三段式。值得注意的隐藏与孤立项：

| 项 | 状态 | 出处 |
|---|---|---|
| `guide/quickstart`（快速搭建体验） | sidebar.ts:13 被注释，但首页 CTA"私有化部署"链接指向它（`src/index.md:19-20`） | 页面存在、侧栏隐藏 |
| `dev/backend/deploy`（tsdd CLI 一键部署） | sidebar.ts:30 被注释 | 页面存在、侧栏隐藏 |
| `dev/backend/k8s` | sidebar.ts:42 被注释；正文仅"欢迎大佬贡献部署脚本"（`k8s.md:5`） | 占位 |
| `dev/backend/config-nginx` | sidebar.ts:47 被注释；**文件 0 行（空文件）** | 空占位 |
| manager 工作台手册（workplace/manage、configuration） | sidebar.ts:209-215 整段注释 | 页面存在、侧栏隐藏 |
| sidebar.ts:238 `"/demo"` 分区指向 `/demo/` | 该路径不存在对应内容 | 疑似残留配置 |

---

## 4. 核心内容走读

### 4a. dev/backend 部署体系

文档提供 **5 种成文 + 1 种隐藏 + 1 种占位**的部署路径：

| 部署方式 | 文件（行数） | 要点与适用场景 |
|---|---|---|
| Docker Compose（**推荐**） | `deploy-compose.md`（247 行） | 完整 compose 清单：wukongim + tangsengdaodaoserver + web + manager + minio + mysql + redis + adminer 共 8 个服务（`deploy-compose.md:32-134`）；`.env` 参数化；适合有 Docker 基础的私有化部署主力路径 |
| tsdd CLI 一键部署 | `deploy.md`（52 行，侧栏隐藏） | `curl install.sh → tsdd install --ip → tsdd start → tsdd doctor`（`deploy.md:18-37`），声明"一键安装程序其实就是采用的docker compose的部署方式"（`deploy.md:46`），配置数据落在 `~/tsdd/`（`deploy.md:48`）；**这是与 TangSengDaoDaoCli 仓库对账的核心入口** |
| 小皮面板 | `xp-panel.md`（100 行） | 面板应用商店一键安装 + 防火墙放行 82/83 端口；面向不熟悉命令行的 Windows/面板用户 |
| 1Panel | `1panel.md`（144 行） | 先在 1Panel 商店装 MySQL/Redis/Minio，再从 `github.com/TangSengDaoDao/appstore` 克隆本地应用安装（`1panel.md:56-60`）；**与 appstore 仓库直接关联** |
| 宝塔 | `btpanle.md`（316 行，最长部署文） | 从 appstore 仓库 `btpanel` 目录导入 Compose 模版（`btpanle.md:39-51`），手工建 `docker-compose.yaml`；compose 内容与其它方式存在镜像版本漂移（见 4a 末尾与第 5 节） |
| Rainbond | `rainbond.md`（122 行） | Rainbond Cloud 应用市场安装，选择 **1.5.1** 版本（`rainbond.md:43`） |
| K8s | `k8s.md`（4 行） | 仅一行"欢迎大佬贡献部署脚本"，纯占位 |

**支撑文档**：`docker-install.md`（Ubuntu 装 Docker）、`port.md`（端口矩阵）、`verify.md`（`/v1/health` 健康检查）、`deploy-mysql.md`（adminer 8306 连 MySQL）、`question.md`（FAQ：GLIBC_2.28、minio 签名错误）。

**全量配置参考的组织方式**（`fullconfig.md`，207 行）：单页 YAML 全量注释式清单，按 16 个区块组织——基础 / 悟空IM / db / 外网 / 日志 / 短信（aliyun、aliyun 国际、uniSMS、smsbao 四家）/ 文件服务（minio、oss、seaweed、qiniu 四家）/ 推送（apns、hms、mi、vivo、oppo、firebase 六家）/ 注册 / 内置账户 / 头像 / 短号 / 机器人 / 第三方登录（gitee、github）/ 缓存。末尾给出"配置转环境变量"规则：`TS_` 前缀 + 层级用 `_` 连接 + 全大写（`fullconfig.md:200-208`），例如 `db.mysqlAddr` → `TS_DB_MYSQLADDR`。**这是与 TangSengDaoDaoServer 的 `modules/config` 包对账的权威清单**。

**架构口径**（`architecture.md:14-22`）：两层架构——通讯层 WuKongIM（长连接/消息收发/存储）+ 业务层 TangSengDaoDao（好友/群组/朋友圈）；WuKongIM 经 **Webhook（GRPC）** 推送聊天数据给业务层，业务层调 WuKongIM 发消息 API 投递系统消息；主要技术栈 `golang, mysql, redis`（`architecture.md:25`）。项目结构文档（`structure.md:18-33`）列出 `internal/api` 下 14 个业务模块（base/channel/common/file/group/message/qrcode/report/robot/source/statistics/user/webhook），可与 Server 仓库实际目录对账。

**版本漂移线索（已确认）**：`deploy-compose.md:94` 使用 `minio:RELEASE.2023-07-18T17-49-40Z`、`adminer:latest`，而 `btpanle.md:130` 使用 `minio/minio:RELEASE.2024-04-06T05-26-02Z`、`btpanle.md:176` 使用 `adminer:4.8.1`——同一文档站内不同部署文的依赖版本不一致。

### 4b. api/ 集成文档

**关键辨析（对账重点）**：`src/api/` 下 7 篇文档描述的是 **WuKongIM 通讯端（5001 端口）的服务端 API**，而非 TangSengDaoDaoServer 业务端（8090 端口）的 API。证据：

- `api/index.md:11`："`注意：所有API是给你们自己的业务系统调用的，不要暴露到外网……（将5001端口外网的访问屏蔽掉）`"——5001 是 WuKongIM 的 API 端口（`port.md:6`）；
- `api/index.md:29`："获取客户端连接**悟空 IM**的地址"；
- 文内的 `/message/send`、`/channel`、`/conversation/sync` 等均为 WuKongIM 原生接口形态；
- 这些页面疑似从 WuKongIM 官方文档（githubim.com）移植：`api/channel.md:8`、`api/conversation.md:8` 中的 `/guide/initialize`、`/guide/fullconfig` 链接指向的是 WuKongIM 文档站的路径结构，在本站不存在对应页面（**移植残留死链，[CONFIRMED]**）。

**API 规则**（`api/index.md:13-25`）：HTTP 200 即成功（部分接口只判状态码）；400 参数错误返回 `{"msg","status"}`；500 服务器错误；文档只列成功响应。**认证方式**：文档未在本区说明认证头，仅 `fullconfig.md:28` 提到 `wukongIM.managerToken`（悟空IM 管理者 token，配置了才需要填）——认证口径不完整，需在 Server/WuKongIM 仓库核实 [PLAUSIBLE]。

**长连接地址获取**（`api/index.md:27-77`）：`GET /route?uid=xxx` 返回 `tcp_addr`/`ws_addr`；`POST /route/batch` 批量获取。

**6 个资源文档覆盖度**：

| 文档 | 行数 | 覆盖端点 |
|---|---|---|
| message.md | 208 | `POST /message/send`、`/message/sendbatch`、`/channel/messagesync`（含双向拉取逻辑详解，`:92-117`）、`/message/sync`、`/message/syncack` |
| user.md | 110 | `/user/token`（注册/登录）、`/user/onlinestatus`、`/user/systemuids_add`、`/user/systemuids_remove`、`/user/device_quit` |
| channel.md | 229 | `/channel`（创建/更新）、`/channel/delete`、订阅者增删、黑/白名单 add/remove/set 共 10 个端点 |
| conversation.md | 110 | `/conversation/sync`（增量同步串格式 `channelID:channelType:last_msg_seq|...`，`:24`）、`/conversations/setUnread`、`/conversations/delete` |
| datasource.md | 109 | 4 个 cmd：`getSubscribers`/`getBlacklist`/`getWhitelist`/`getSystemUIDs`；说明"数据源只有在重启后第一次启动时才会加载"（`:10`） |
| webhook.md | 93 | 3 个事件：`user.onlinestatus`（设备级在线状态，数据格式 `[uid-设备标识-在线状态-连接ID-...]`，`:30-39`）、`msg.offline`（离线推送）、`msg.notify`（全量消息，默认 500ms 批量推送 `webhook.msgNotifyEventPushInterval`，`:71`） |

**TSDD 业务端（8090）自有 API 在本站几乎无文档**：仅出现 `/v1/ping`（健康检查探针，`deploy-compose.md:52`）、`/v1/health`（`verify.md:5`）、`/v1/datasource`（Webhook 数据源回调节点，`deploy-compose.md:179`）三个端点；`dev/backend/api.md` 正文只有"待补充"二字（`api.md:2`），sidebar 把 API 文档外引至 `https://apidocs.botgate.cn/`。**业务端 API 的权威文档不在本仓库**，这是覆盖度最大的缺口。

### 4c. dev/ 各端开发文档完整度对比

| 端 | 篇目（行数） | 深度评估 |
|---|---|---|
| **Android** | intro(14)、build(62)、config(33)、structure(37)、module(110)、aar_import(39)、code_import(33)、question(5) | **四端中最完整**：模块化注入机制（`WKLoginApplication.getInstance().init(this)`，`intro.md:3`）、改名/改包名/换 Logo/签名打包全流程截图（`build.md:26-62`）、compileSdk 33/minSdk 23 徽章（`build.md:10-13`）、aar 与源码两种引入方式；FAQ 是占位（"以后会整理到这里"） |
| **iOS** | intro(15)、build(24)、config(15)、structure(13)、module(157)、moduleimport(26)、package_notes(20)、question(18) | **中等**：模块化 pod 机制清晰（`intro.md:3`、`module.md:5-13`），但 build 仅 `pod install` + `open xcworkspace` 三步（`build.md:9-24`），无签名/上架指引；`package_notes.md` 只有 dev_team 与 frameworks.sh 补丁两条；FAQ 仅 1 条 Aspects 框架问题 |
| **Web/PC** | intro(15)、build(117)、config(12)、structure(14)、module(2，仅"待补充")、moduleimport(52)、question(15) | **中等偏弱**：build 的技术栈徽章最全（React 17.0.2 / TS 5.0.4 / Webpack 5.88.2 / Electron 26.0.0 / Semi UI 2.24.2 / WukongIM JS SDK 1.2.10 等，`build.md:6-35`），含 web/electron 三平台打包命令；但**模块开发正文为空**（`module.md:1-2` "待补充"），structure 仅 14 行 |
| **Manager（后台管理）** | intro(7)、build(128)、config(68)、structure(44)、question(1，仅标题)、user-manual/（19 篇） | **两极分化**：开发文档单薄（无 module 开发文），但**用户手册是全站最成体系的产品文档**——通用功能/首页仪表盘/用户(新增/列表/封禁)/群组(列表/封禁)/消息(记录/违禁词)/举报(用户/群聊)/工具(APP升级)/设置(通用/改密码) 共 19 篇带截图的操作手册；技术栈声明 Vue3.3 + Vite5 + Pinia + Element-Plus（`build.md:3`） |

后端另有 `moduleimport.md`（28 行）：以朋友圈模块 `moments` 为例，模块复制到 `modules/` + 在 `internal/modules.go` 空白导入（`moduleimport.md:16-24`）——**这是收费模块分发的安装机制说明，与 price/ 售卖页形成闭环**。

### 4d. guide / price：产品化程度

**定价模型**（`price/index.md`）：

- 模块化售卖：基础模块开源免费商用（`index.md:34-39`），12 个收费模块分"非源码/源码"两档价格（收藏 0.1w/0.2w → 音视频 1.8w/3.6w，`index.md:41-123`）；
- 总计非源码 7.88w、源码 15.76w（`index.md:126-129`）；打包优惠 15.5w 送两个月技术服务（`index.md:143`）；
- 增值服务：一对多 5x8 技术咨询 2999 元/月、每月限 10 人（`index.md:149-151`）；
- 售后口径："一次购买免费升级……期限为合同签订后一年内"（`index.md:135`）、"所有功能以官方提供的在线演示功能为准"（`index.md:137`）；
- 明细页 `price/detail.md`（1054 行，全站最长）逐模块列功能表。
- **锚点 bug [CONFIRMED]**：`index.md:66` 标签模块的"明细"链接指向 `#文件模块`；`index.md:108` 客服模块的"明细"链接指向 `#旗舰模块`——复制粘贴错误。

**产品化要素**：在线 Demo（`demo.md:14` Web 端 `https://web.botgate.cn`，Win/Mac 安装包托管于 `file.githubim.com`）、5 个公开演示账号（`demo.md:23-27`）、隐藏注册入口彩蛋（`quickstart.md:19` 登录页长按标题、默认短信验证码 123456）、微信私域引流（guide/index.md:59、support.md:18、price/index.md:162 三处微信二维码）、竞品对比表（`guide/index.md:21-27`，对比网易云信/融云/环信/腾讯云IM）。整体呈现"开源引流 + 模块收费 + 服务收费"的成熟商业化形态。

---

## 5. 对外接口：文档所描述的接口体系地图

文档站自身不暴露 API；本节汇总文档中记录的端口/服务/端点清单，作为与 Server / WuKongIM / Cli / appstore 仓库对账的基准。

### 5.1 端口矩阵

| 端口 | 归属 | 说明 | 对外开放 | 出处 |
|---|---|---|---|---|
| 8090 | TangSengDaoDaoServer | 业务 API（客户端调用），`/v1/*` | 是 | `port.md:5`；`fullconfig.md:14` |
| 5001 | WuKongIM | 通讯端 HTTP API（业务端调用） | **否（安全隐患）** | `port.md:6`；`api/index.md:11` |
| 5100 | WuKongIM | APP TCP 长连接 | 是 | `port.md:9` |
| 5200 | WuKongIM | Web Websocket | 是 | `port.md:10` |
| 5300 | WuKongIM | 监控端口 | compose 中映射（`deploy-compose.md:40`），port.md 未列 | `1panel.md:101` |
| 82 | TSDD Web 端 | nginx 容器 80→82 | 是 | `port.md:7` |
| 83 | TSDD 后台管理 | nginx 容器 80→83 | 是 | `port.md:8` |
| 9000 | minio | 文件服务（客户端访问） | 是 | `port.md:11` |
| 9001 | minio | 控制台 | 否 | `port.md:12` |
| 8306 | adminer | MySQL Web 管理（调试） | 否（"为了安全生产不要打开" `deploy-compose.md:131`） | `port.md:13` |
| 6979 | TangSengDaoDaoServer | Webhook gRPC 监听（供 WuKongIM 回调） | 内网 | `fullconfig.md:15`；`deploy-compose.md:177` |
| 6379 / 3306 | redis / mysql | 基础设施，仅容器网络内 | 否 | `deploy-compose.md:197`、compose 服务名寻址 |

### 5.2 端点清单（文档中出现的全部 HTTP/gRPC 端点）

**WuKongIM（5001）**——`src/api/`：见 4b 节表格，共约 20 个端点 + datasource 4 cmd + webhook 3 事件。

**TangSengDaoDaoServer（8090，文档中仅零散出现）**：

| 端点 | 用途 | 出处 |
|---|---|---|
| `GET /v1/ping` | compose 健康检查 | `deploy-compose.md:52`、`btpanle.md:90` |
| `GET /v1/health` | 部署验证，返回 `{"status":"up"}` | `verify.md:5-11` |
| `POST /v1/datasource` | WuKongIM 数据源回调地址（`WK_DATASOURCE_ADDR`） | `deploy-compose.md:179`、`btpanle.md:214` |
| gRPC `0.0.0.0:6979` | webhook 推送接收（`grpcAddr` / `WK_WEBHOOK_GRPCADDR`） | `fullconfig.md:15`、`deploy-compose.md:177` |

**外部依赖服务（文档点名）**：短信 4 家、对象存储 4 家、推送 6 家（4a 节）、`api.dicebear.com` 默认头像（`deploy-compose.md:203`，备用 `robohash.org`）、gitee/github OAuth（`fullconfig.md:157-166`）。

### 5.3 版本口径清单（对账专用）

| 版本号 | 对象 | 出处 |
|---|---|---|
| **1.1.2** | 文档站首页展示的产品 release tag（硬编码） | `.vitepress/utils/fetchReleaseTag.js:2` |
| wukongim:**v2** | 通讯端镜像 tag | `deploy-compose.md:34`、`btpanle.md:70` |
| tangsengdaodaoserver:**v1.5** | 业务端镜像 tag | `deploy-compose.md:48`、`btpanle.md:85` |
| tangsengdaodaoweb:**latest** / tangsengdaodaomanager:**latest** | Web/管理端镜像 | `deploy-compose.md:74,84` |
| minio **RELEASE.2023-07-18T17-49-40Z** | compose 部署文 | `deploy-compose.md:94` |
| minio **RELEASE.2024-04-06T05-26-02Z** | 宝塔部署文（与上行不一致） | `btpanle.md:130` |
| mysql **8.0.33** | 两处一致 | `deploy-compose.md:113`、`btpanle.md:151` |
| redis **7.2.3** | 两处一致 | `deploy-compose.md:124`、`btpanle.md:166` |
| adminer **latest** vs **4.8.1** | 两篇部署文不一致 | `deploy-compose.md:132` vs `btpanle.md:176` |
| **1.5.1** | Rainbond 应用市场安装版本 | `rainbond.md:43` |
| compileSdkVersion **33** / minSdkVersion **23** | Android | `android/build.md:10-13` |
| React **17.0.2** / TypeScript **5.0.4** / Yarn **1.22.17** / Node **18.17.1** / Webpack **5.88.2** / Electron **26.0.0** / electron-builder **24.9.1** / Semi UI **2.24.2** / turbo **2.0.9** / WukongIM JS SDK **1.2.10** | Web/PC 端技术栈徽章 | `web/build.md:6-35` |
| Vue **3.3** / Vite **5** / Pinia / Element-Plus | 管理端技术栈 | `manager/build.md:3` |
| node:18.0.0 / nginx:latest | 文档站自身镜像 | `Dockerfile:1,12` |
| vitepress **1.0.2** | 文档引擎 | `package.json:23` |
| Ubuntu 22+（一键脚本建议）；GLIBC_2.28 | 系统要求 | `deploy.md:8`、`question.md:6-19` |
| 文档站镜像 `registry.cn-shanghai.aliyuncs.com/wukongim/tangsengdaodaodocs:latest` | 本站发布物 | `Makefile:5-6` |

---

## 6. 部署与运维（文档站自身）

- **Dockerfile**（两阶段）：
  - 阶段一 `node:18.0.0`：curl 安装 yarn → `yarn install` → `yarn build`（产物在 `.vitepress/dist`）。**注意**：用 yarn 而非 packageManager 声明的 pnpm；`COPY package.json .` + `COPY yarn.lock .` 分层缓存合理，但 pnpm-lock.yaml 未参与构建。
  - 阶段二 `nginx:latest`：拷贝 `docker-entrypoint.sh`（改名 docker-entrypoint2.sh）、`nginx.conf.template`、dist 产物；ENTRYPOINT 先跑 envsubst 再启 nginx。
- **envsubst 疑点 [CONFIRMED]**：`docker-entrypoint.sh:5` 执行 `envsubst '${API_URL}' < /nginx.conf.template > /etc/nginx/conf.d/default.conf`，但 `nginx.conf.template` 全文**没有任何 `${API_URL}` 占位符**（仅 80 端口静态托管 + `try_files` SPA fallback）。该 envsubst 是无效操作，模板疑似从 Web 端仓库（需要 `API_URL` 反代）复制后未清理。功能上无害，但属于死代码。
- **Makefile**：`make build`（`docker build -t tangsengdaodaodocs --platform linux/amd64`）；`make deploy` 推送到 `registry.cn-shanghai.aliyuncs.com/wukongim/tangsengdaodaodocs:latest`——与全生态统一的阿里云上海镜像仓库口径一致。
- **本地开发**：`pnpm run dev` / `build` / `serve`（README.md:5-19）；`engines: node ^14.18 || >=18`。
- **无 CI**：仓库无 .github/workflows 等自动化，发布靠手工 `make deploy`。

---

## 7. 安全与合规审查

### 7.1 泄露性信息

| 级别 | 发现 | 出处 |
|---|---|---|
| **[CONFIRMED]** | 第三方客服 widget **明文 live API key**：`https://widget.tgo-ai.com/tgo-widget-sdk.js?api_key=ak_live_7Lh6h82EoXh9lL4lrqCZjuxfHzv5DX6n`。前端 key 天然公开，但 `ak_live_` 前缀表明是生产密钥，若该服务支持后台 API 调用则存在滥用面（提交记录显示为 "add tgo cs" 引入） | `.vitepress/config.ts:22` |
| **[CONFIRMED]** | 百度统计 ID 明文 `945767791d09113d72014cd801857369`（低风险，统计 ID 本就公开） | `.vitepress/config.ts:21` |
| **[CONFIRMED]** | 部署文档默认凭据全套：`MYSQL_ROOT_PASSWORD=Aa1234567`、`MINIO_ROOT_PASSWORD=Aa1234567`、`WK_JWT_SECRET=wk_secret_123`、`TS_ADMINPWD=admin1234567`（用户名 superAdmin）、`TS_SMSCODE=123456`。文档有提示"建议修改"（`deploy-compose.md:212-222`），但 `.env` 示例即复制即用，用户照抄不改的风险高；尤其 `TS_SMSCODE=123456` 意味着**默认万能短信验证码** | `deploy-compose.md:159,168,187,199,205` |
| **[CONFIRMED]** | 公开演示账号密码 5 组（15900000002~06 / a1234567） | `demo.md:23-27` |
| **[CONFIRMED]** | 隐藏注册入口与默认验证码 123456 写进公开文档（`quickstart.md:19`），与上条叠加，私有化部署若不改 `TS_SMSCODE` 可被任意注册 | `guide/quickstart.md:17-19` |
| **[CONFIRMED]** | 宝塔安装脚本带推广/渠道参数 `ed8484bec`（官方 install 脚本的推广码） | `btpanle.md:9-21` |
| **[CONFIRMED]** | 文档中未发现真实生产 IP/域名/数据库密码泄露；示例均为 `xx.xx.xx.xx` 占位。出现的真实域名均为公开服务：`tsdaodao.com`、`botgate.cn`（`web.botgate.cn`、`api.botgate.cn`、`apidocs.botgate.cn`）、`githubim.com`、`file.githubim.com` | 全文检索 |
| **[PLAUSIBLE]** | `manager/config.md:18` 开发环境 API 指向 `https://api.botgate.cn/v1/`（官方演示后端），开发者二开时默认连官方服务器，存在数据流向官方的隐患（隐私口径） | `dev/manager/config.md:15-20` |
| **[PLAUSIBLE]** | 5001 端口屏蔽告诫只在 `api/index.md:11`（导航孤岛页）和 `port.md:6` 出现，主部署文 `deploy-compose.md:37` 中 5001 映射行已注释掉（默认安全），做法正确；但宝塔/1Panel 文档未重复强调 | `deploy-compose.md:37` |

### 7.2 合规内容

- **法律声明**（`guide/legal.md`，仅 7 行）：3 条免责条款——软件仅用于合法用途、用户行为不可控不担责、违规用户承担全部法律责任并停止服务。**没有隐私政策、用户协议、数据处理说明**；对一款处理即时通讯内容（含手机号注册）的产品，合规文档明显单薄 [PLAUSIBLE 风险]。
- **版权与备案**：页脚声明版权归"上海信必达网络科技有限公司"（`config.ts:58`），挂 `沪ICP备2021032718号-3` 与 `-5` 双备案号（`config.ts:59`）。
- **开源协议口径**：多处声明 Apache-2.0 可商用（`guide/index.md:9,53`；`src/index.md:25`），与 LICENSE 文件一致。
- **下载页链接完整性**：`demo.md:14` 的 Win/Mac 包（`file.githubim.com/download/TangSengDaoDao-latest.{exe,dmg,dmg}`）为外链，无法离线验证有效性（**待验证**）；Android/iOS 以二维码图片提供，无直链；iOS 引导至 App Store 搜索。

---

## 8. 质量评估

### 8.1 文档覆盖度矩阵（功能 × 端）

| 功能区 | backend | android | ios | web | manager | guide/api |
|---|---|---|---|---|---|---|
| 介绍/架构 | ✅ architecture+structure | ✅ | ✅ | ✅ | ✅ intro | ✅ guide/index |
| 编译/构建 | ✅（部署文替代） | ✅ 详尽 | ⚠️ 极简（3 步） | ✅ 详尽 | ✅ | — |
| 配置 | ✅ fullconfig 权威 | ✅ | ✅ | ⚠️ 仅服务端地址 1 项 | ✅ | — |
| 项目结构 | ✅ 14 模块清单 | ✅ | ⚠️ 13 行 | ⚠️ 14 行 | ✅ | — |
| 模块开发 | ⚠️ `module.md` **空文件**；`api.md` "待补充" | ✅ 110 行 | ✅ 157 行 | ❌ `module.md` "待补充" | ❌ 无 | — |
| 模块引入 | ✅ | ✅（aar+源码两种） | ✅ | ✅ | — | — |
| 部署/打包 | ✅ 5+1 种方式 | ✅ 签名打包 | ⚠️ package_notes 两条 | ✅ web+三桌面端 | ✅ build | — |
| API 文档 | ❌ 业务端 API 缺失（外链 apidocs.botgate.cn） | — | — | — | — | ⚠️ api/ 实为 WuKongIM API |
| FAQ | ⚠️ 2 条 | ❌ 占位 | ⚠️ 1 条 | ⚠️ 1 条 | ❌ 空 | ❌ guide/question 占位 |
| 用户手册 | — | — | — | — | ✅ 19 篇成体系 | ✅ demo/download |
| K8s 部署 | ❌ 占位 | — | — | — | — | — |

### 8.2 中英文情况

全站 zh-CN 单语言（`config.ts:14`），无英文版；代码注释、配置项为英文。面向海外用户的私有化部署场景存在语言门槛。

### 8.3 死链 / 过时线索

| 级别 | 发现 |
|---|---|
| [CONFIRMED] | `ignoreDeadLinks: true`（`config.ts:15`）掩盖死链，构建不报错 |
| [CONFIRMED] | `api/channel.md:8`、`api/conversation.md:8,10`、`api/message.md:33` 引用 `/guide/initialize`、`/guide/fullconfig`、`/guide/proto`——本站无此页面，系从 WuKongIM 文档移植的残留 |
| [CONFIRMED] | `sidebar.ts:238` 注册 `/demo` 侧栏分区指向不存在的 `/demo/` |
| [CONFIRMED] | `price/index.md:66,108` 两处"明细"锚点指错（标签→文件模块、客服→旗舰模块） |
| [CONFIRMED] | 首页版本号 1.1.2 硬编码（`fetchReleaseTag.js:2`），函数名 `fetchReleaseTag` 暗示本应动态拉取 release，实为写死——与代码仓库 release 漂移的高危点 |
| [CONFIRMED] | minio / adminer 镜像版本在两篇部署文之间不一致（见 5.3 表） |
| [PLAUSIBLE] | `guide/fastentry.md:21` 把 WuKongIM 的文档链接写成 `githubim.com/sdk/android.html`（SDK 文档而非主项目文档），口径粗糙 |
| [PLAUSIBLE] | `support.md:8` 错别字 "Adnroid" |
| 待验证 | `demo.md` 全部 `file.githubim.com` 下载/截图外链、`web.botgate.cn` 在线 Demo 的可达性 |

### 8.4 篇幅分布（82+1 篇，5312 行）

- **头部重内容**（>200 行）：`price/detail.md` 1054、`btpanle.md` 316、`deploy-compose.md` 247、`api/channel.md` 229、`api/message.md` 208、`fullconfig.md` 207。
- **尾部空心内容**（≤2 行）：`backend/config-nginx.md`（0 行）、`backend/module.md`（0 行）、`backend/api.md`（"待补充"）、`web/module.md`（"待补充"）、`manager/question.md`（仅标题）、`guide/question.md`（占位）、`manager/user-manual/workplace/*`（各 1 行，侧栏已隐藏）、`backend/k8s.md`（占位）。
- 整体呈"部署与售卖内容扎实、二开深度内容不均、FAQ 普遍荒废"的分布。估算有效内容率：约 8 篇为占位/空文档（≈10%）。

---

## 9. 风险与改进建议

| # | 风险/问题 | 级别 | 建议 |
|---|---|---|---|
| 1 | 业务端（8090）API 文档缺失，权威 API 文档外引至 apidocs.botgate.cn 独立站点，本站 `dev/backend/api.md` 仅"待补充"；用户极易把 `src/api/`（WuKongIM 5001 API）误认为唐僧叨叨业务 API | **高（口径混淆）** | 在 `src/api/index.md` 顶部加显著提示"以下为 WuKongIM 通讯端 API"；将业务端 API 文档纳入本仓或显式互链 |
| 2 | `src/api/` 分区为导航孤岛（navbar/sidebar 均无入口），只能靠直接 URL 访问 | 中 | 在 navbar 或 backend sidebar 注册入口，或移除该分区避免口径分裂 |
| 3 | 默认凭据+万能短信验证码 123456 写进可复制示例 | 高（安全） | 示例密码改为占位符 `<随机生成>`；部署文增加"上线前安全清单"章节 |
| 4 | tgo-ai widget live key、宝塔推广码入库 | 低-中 | 确认 key 作用域；推广码加注释说明 |
| 5 | 版本口径多点硬编码且互不一致（首页 1.1.2 / 镜像 v1.5 / Rainbond 1.5.1 / minio 两版本） | 中（drift） | 建立单一版本来源（如 frontmatter 或构建期注入），部署文间共享 include |
| 6 | pnpm 声明 vs Dockerfile 用 yarn，双 lockfile 并存 | 低 | 统一为 pnpm（`corepack`），删除 yarn.lock |
| 7 | nginx 模板 envsubst `${API_URL}` 为无效死代码 | 低 | 删除 envsubst 步骤或在模板中加入 API 反代 |
| 8 | 空文档/占位文档约 10%（k8s、config-nginx、backend/module、web/module、各端 FAQ） | 中 | 排期补全或从侧栏摘除（部分已隐藏但文件仍在，可被搜索引擎索引） |
| 9 | legal 仅 3 条免责，无隐私政策/用户协议 | 中（合规） | 补充隐私政策（手机号、消息内容处理）、开源模块商用授权边界说明 |
| 10 | `ignoreDeadLinks: true` + 多处已确认死链/错锚 | 中 | 关闭该开关或接入链接检查 CI |
| 11 | 单语言（仅中文） | 低 | 视海外推广需求评估 i18n |
| 12 | manager 开发文档薄（无模块开发）、workplace 手册雪藏 | 低 | 与 Manager 仓库迭代同步补齐 |

---

## 附录：分析方法与置信度说明

- 全部 83 篇 markdown 均经 `wc -l` 统计；关键论断均经 Read 工具逐行确认后引用（file:line）。
- `api/` 分区归属（WuKongIM 而非 TSDD 业务端）系依据 5001 端口、端点形态与移植残留链接综合判定，标记为待与 WuKongIM 仓库代码最终核对。
- 外部链接（下载、Demo、apidocs）的可达性未做联网验证，统一标"待验证"。
