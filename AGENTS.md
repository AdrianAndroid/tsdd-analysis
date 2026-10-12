# AGENTS.md — AI 协作者工作指南

本文件面向在本仓库中工作的 AI 编码/分析代理（Claude Code、Codex 等）。读完后再动手。

## 项目性质

这是一个**源码分析工程**，不是可运行软件。产出物是 `docs/` 下的中文分析文档；`projects/` 下的 13 个 git submodule 是被分析的源码（只读）。

## 目录约定

| 路径 | 用途 | 可否修改 |
|---|---|---|
| `docs/` | 全部分析产出（MkDocs 源目录） | ✅ 主要工作区 |
| `projects/<仓库>/` | 被分析的源码，submodule 固定在分析基线版本 | ❌ 只读，禁止在其中改代码、切分支、升级版本 |
| `mkdocs.yml` / `requirements.txt` | 站点构建配置 | ⚠️ 仅在有明确需求时改 |
| `README.md` / `AGENTS.md`（根目录） | 项目说明 / 本文件 | ⚠️ 结构变化时同步更新 |

## 文档铁律（历史用户指令，必须遵守）

1. **一切结论基于真实源码**。关键论断必须附证据，格式为 `相对路径:行号`（相对所在仓库根，如 `modules/webhook/api.go:104`）。禁止编造类名/方法名/字段/协议字段。
2. 源码缺失处明确标注 **【缺少源码，无法确认】**，不得猜测补全。
3. 发现文档/注释与代码矛盾时，**指出矛盾并同时给出两处代码位置**。
4. 调用栈必须真实（`类A.方法B → 类C.方法D`），从源码逐级跟踪得出。
5. 文档语言：**中文**；术语保持专业，避免空泛评价。
6. 每个分析章节：**先 Mermaid 图 → 再文字分析 → 最后源码证据**。
7. 引用行号会随上游版本漂移——引用前用 `grep`/`sed` 抽查确认行号仍然有效（submodule 已固定版本，正常情况下稳定）。
8. 新增/修改文档后，更新 `docs/README.md` 索引与 `mkdocs.yml` 的 `nav`。

## 文档组织

- 每仓库一份 `docs/<仓库名>.md`，统一模板：概述 → 技术栈 → 架构（Mermaid）→ 核心流程走读 → 对外接口 → 部署运维 → 安全审计 → 质量评估 → 风险建议。
- 跨仓库专题独立成文（现有范例：`docs/IM消息机制深度分析.md`，八部分结构）。
- `docs/README.md` 是总览索引（MkDocs 首页），只放索引、关系图、对账结论，不放单仓库细节。

## 构建验证

改完文档后本地验证站点可构建（需要 Python 环境）：

```bash
pip install -r requirements.txt
mkdocs build --strict   # 通过才算完成；链接断裂/nav 缺项会报错
```

`site/` 是构建产物，已 gitignore，不要提交。

## Git 工作流

- 分支：`main`；远程：`git@github.com:AdrianAndroid/tsdd-analysis.git`。
- 提交粒度：一个专题/一次结构调整为一次 commit；commit message 用中文简述产出。
- **不要**执行 `git submodule update --remote`（会把分析基线版本顶到上游最新，使文中行号证据失效）。
- 新增源码仓库一律以 submodule 加入 `projects/`，并设 `shallow = true`，在对应文档中注明签出版本。
- 仓库身份配置：`git config user.name/user.email` 沿用上次提交的作者（repo 本地配置）。

## 已知坑

- `git mv` 移动 submodule 时 git 会自动改写 `.gitmodules` 的 `path`，不要再手动 sed 追加前缀。
- MkDocs Material 渲染 Mermaid 依赖 `pymdownx.superfences` 的 `custom_fences` 配置（已配好）；新增图表直接用 ```` ```mermaid ```` 代码块即可。
- 文档文件名含中文（如 `IM消息机制深度分析.md`），MkDocs 支持，但新增文件时确保 `nav` 中路径与实际文件名完全一致（UTF-8）。
