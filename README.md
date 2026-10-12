# tsdd-analysis — TangSengDaoDao 生态源码分析

对开源 IM 项目 **唐僧叨叨（TangSengDaoDao）** 及其底层 **WuKongIM** 生态共 13 个源码仓库的深度分析工程。所有分析结论均基于真实源码，关键论断附 `文件路径:行号` 证据，可逐条复核。

## 目录结构

```
tsdd-analysis/
├── README.md            # 本文件
├── AGENTS.md            # AI 协作者（Claude Code 等）工作指南
├── mkdocs.yml           # MkDocs 站点配置
├── requirements.txt     # Python 依赖（mkdocs + material 主题）
├── docs/                # 全部分析文档（MkDocs 源目录）
│   ├── README.md        #   总览索引：13 仓库关系、对账结果、共性风险
│   ├── IM消息机制深度分析.md   # 专题：IM 协议/收发链路/离线/多端同步工程评审
│   └── <仓库名>.md      #   每个仓库一份独立深度分析
└── projects/            # 13 个源码仓库（git submodule，版本已固定）
    ├── TangSengDaoDaoServer/       # 业务服务端（Go）
    ├── TangSengDaoDaoServerLib/    # 服务端框架库（Go）
    ├── TangSengDaoDaoWeb/          # Web/桌面端（React + Electron）
    ├── TangSengDaoDaoAndroid/      # Android 端
    ├── TangSengDaoDaoiOS/          # iOS 端（含 vendor IM SDK）
    ├── TangSengDaoDaoManager/      # 运营管理台（Vue 3）
    ├── TangSengDaoDaoCli/          # 一键部署 CLI（Go）
    ├── appstore/                   # 面板应用商店打包（YAML）
    ├── TangSengDaoDaoJSBridge/     # H5↔原生桥库（TS）
    ├── TangSengDaoDaoDocs/         # 官方文档站（VitePress）
    ├── WuKongIM/                   # IM 协议服务端 v1.2.6（Go，分析基线）
    ├── WuKongIMAndroidSDK/         # Android IM SDK 1.5.0（分析基线）
    └── wukongimjssdk/              # JS IM SDK v1.2.11（分析基线）
```

## 文档索引

| 文档 | 内容 |
|---|---|
| [docs/README.md](docs/README.md) | **总览**：13 仓库定位索引、生态架构与仓库关系、跨仓库对账、共性风险汇总 |
| [docs/IM消息机制深度分析.md](docs/IM消息机制深度分析.md) | **专题评审**（八部分）：WuKongIM 二进制协议、HTTP 与长连接分工、移动端消息监听注册与收发全链路、离线消息/ACK/多端同步、风险与优化建议、待确认清单 |
| [docs/TangSengDaoDaoServer.md](docs/TangSengDaoDaoServer.md) 等 10 份 | 每个仓库一份：架构与模块、核心流程代码级走读、部署运维、安全审计、风险建议 |

## 如何运行（生成文档站点）

本项目文档使用 [MkDocs](https://www.mkdocs.org/) + [Material 主题](https://squidfunk.github.io/mkdocs-material/) 构建，支持 Mermaid 图表渲染与中文搜索。

### 1. 克隆仓库（含子模块）

```bash
git clone --recursive git@github.com:AdrianAndroid/tsdd-analysis.git
cd tsdd-analysis

# 已克隆但未拉子模块时：
git submodule update --init
```

> 注意：文档**生成站点不需要**子模块；子模块（`projects/` 下约 13 个源码仓库）仅在需要对照源码验证文中 `file:line` 引用时才必须拉取。

### 2. 安装依赖（建议虚拟环境）

```bash
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 3. 本地预览（热重载）

```bash
mkdocs serve
```

浏览器打开 <http://127.0.0.1:8000>，编辑 `docs/` 下任意 `.md` 会自动刷新页面。

### 4. 构建静态站点

```bash
mkdocs build        # 产物输出到 site/ 目录（已 gitignore）
```

`site/` 为纯静态文件，可直接部署到任意静态托管（Nginx / GitHub Pages / Vercel 等）。例如 GitHub Pages：

```bash
mkdocs gh-deploy    # 构建并推送到 gh-pages 分支
```

## 分析基线说明

- **TangSengDaoDao 组织 10 仓库**：克隆时的 main/master 分支（submodule 已固定 commit）。
- **WuKongIM 组织 3 仓库**（专题分析时补充，版本与部署基线对齐）：`WuKongIM` v1.2.6、`WuKongIMAndroidSDK` 1.5.0、`wukongimjssdk` v1.2.11。
- `projects/` 下所有仓库为**只读分析对象**，请勿在其中修改代码；分析产出只写入 `docs/`。
