# 名称：DSH 版本号升级 · 0.1.1 → 0.1.2

本技能包的名字是**「DSH 版本号升级（0.1.1 → 0.1.2）」**（仓库：
`dsh-version-upgrade-0.1.1-to-0.1.2`）。命名即版本范围：**从哪个 DSH 版本
升级到哪个版本**，一目了然。DSH 每升一个版本，就可以按同一套路再生成一个新
技能包（例如 `dsh-version-upgrade-0.1.2-to-0.1.3`）；存量插件作者只需要
挑对应自己目标版本的包。

它是一份面向**第三方 DeepSeek Harness 插件作者**的 Agent 技能集合包：把插件
repo 里的代码与清单从 DSH **0.1.1**（`dsh-v0.1.1-rc.1/rc.2`，2026-08-21）
升级到 **0.1.2-alpha.1**（2026-08-28），覆盖 manifest 契约、host 半身、
client 半身与分发验收四个面。

> 一句话：官方文档教插件作者「怎么写」，这套技能教插件作者「怎么把老代码
> 升级过 0.1.1 → 0.1.2 这道坎」——全部内容来自对 DSH 官方 git 历史的逐条核对
> 与一次真实插件规模化迁移的实战踩坑。

## 它解决什么问题

DSH 的版本号每上一个台阶（0.1.1 → 0.1.2），插件作者面临的不是「改一个
版本号」，而是一整套契约迁移：清单字段、构建产物、host 服务面、client
槽位与调用渠道都可能变。本技能包把这套「跟着 DSH 版本号升级插件」的经验
固化成了可复用的技能。

## 为什么做这套技能

- DeepSeek Harness 在 `dsh-v0.1.1-rc.1` 与 `dsh-v0.1.2-alpha.1` 之间是
  **一次大重构**（约 1100+ commits）：web 客户端重写、浏览器侧服务面 Remote 化、
  `code-mode` → `ptc` 改名、槽位系统改声明制等。
- 官方文档（`docs/cordis-tutorial/`、`docs/cookbook/`、`docs/subsystems/`）
  讲的是「怎么写一个新插件」，**没有「已有插件怎么升级」的存量迁移文档**；
  而升级过程中大量故障的特征恰恰是**静默**：入口消失、槽位不渲染、数据空白、
  装完启动失败但源码目录一切正常。
- 本包基于一次真实的规模化迁移整理：一个大型 DSH webui 全家桶插件（DSH 0.1.1
  时代产物，十余个功能模块）在 2026-08-28 卸载后拆分为若干独立插件（用量/技能/
  记忆三合一、对话完成胶囊、提示词优化等），部分又融合为聚合插件（供应商中心），
  全程按 0.1.2 现行契约重写并发布。

## 包结构（4 个技能）

| 技能 | 定位 | 内容 |
| --- | --- | --- |
| [`dsh-upgrade-plugin-011-to-012/`](dsh-upgrade-plugin-011-to-012/SKILL.md) | **主入口（必读）** | 迁移五步路线（盘点 → manifest → host → client → 验收）、「症状 → 根因 → 修法」速查表、官方文档地图、完成定义（DoD） |
| [`dsh-upgrade-manifest-contract/`](dsh-upgrade-manifest-contract/SKILL.md) | 清单与构建契约 | package.json 五件套（`main`/`exports`/`files`/`dsh.bundle.patch`/`dsh.client`）、bundle patch 自动注册替代手工 insert 的清理规则、双 bundle 构建（host ESM 自包含 + client CJS factory）、产物随仓库提交且不带 `prepare`（pnpm ≥ 10 拦截）、`files` 精简与换行符、安装失败三种常见原因、已安装位置验收四步 |
| [`dsh-upgrade-host-half/`](dsh-upgrade-host-half/SKILL.md) | host 半身 | **解析面大坑**：`@deepseek-ai/*` 只发源码不发产物（实测 247 个包仅 82 个有 `lib/index.js`），已安装位置裸解析必炸；vendor 化叶子模块三条件；`assertHostExternals` 构建守卫（双正则 + 自测）；`@Remote`/`TypertRemoteService` 服务面与行级 inject 红线；`code-mode` → `ptc` 改名；路由前缀独立；多模块 try/catch 挂载 |
| [`dsh-upgrade-client-half/`](dsh-upgrade-client-half/SKILL.md) | client 半身 | 0.1.2 槽位**声明制**（`ctx.slots.register` 对象形态、四种 cardinality / 三种 scope、组件四 shares props、hooks 五座、业务组件永不见 ctx）；**旧 unary API → Remote 命名空间全量迁移对照表**（16 项：`session.rename`→`sessionTitle/rename`、`llm.models`→`session/modelCatalog`…）；`dsh.client.inject` 模块表真相（`dsh-client-schema-form` 不在表内 → `missed the module table`）；web 重构对挂载点影响（grep 实证、MutationObserver 兜底、React 18 渲染期异常卸载整树、`createPortal` 到 body、ErrorBoundary 包面板）；插件间并存与数据契约延续 |

技能之间按描述里的触发词互相路由：主入口是唯一必读，其余三个按迁移面取用。

## 安装（任一方式）

每个技能目录都是独立的标准 DSH 技能（`SKILL.md` + frontmatter）：

- **官方仓库**：整体放入仓库 `.agents/skills/`（与 `dsh-doc`、`dsh-translate-docs`
  等官方技能并列）；
- **本机使用**：复制技能目录到 `$DSH_HOME/skills/<技能名>/`；
- **按需取用**：只取需要的那一个技能目录。

技能加载后，描述里的触发词会使之在「插件升级 / 迁移 / bundle 契约 / 槽位失效」
等场景自动被选中。

## 内容质量与验证依据

本包每条规则都可溯源，不凭印象：

- **版本差异**：逐条核对 `dsh-v0.1.1-rc.1 / rc.2 → dsh-v0.1.2-alpha.1` 的
  git diff（commit、`packages/`、`tsconfig` 等）；
- **官方权威文档**：`docs/subsystems/extensions.md`、`docs/subsystems/slots.md`、
  `docs/subsystems/web-client.md`、`packages/client/AGENTS.md`；架构笔记
  `.agents/notes/implemented/architecture/` 下的 `profile-plugin-bundles`、
  `unary-apiproxy-remote-migration`、`client-plugin-loading-model`、
  `client-shells-and-dynamic-packages`、`plugin-owned-settings-surface`；
- **实战经验**：2026-08 的真实插件迁移（WebUI 全家桶拆分 / 融合），全部以
  「症状 → 根因 → 修法」形式记录——这些是官方文档不写、但迁移必踩的部分。

> 注：技能文件中出现的 `docs/…`、`packages/…`、`.agents/notes/…` 相对路径
> 均指向 DeepSeek Harness 官方仓库
> （<https://github.com/deepseek-ai/deepseek-harness>）。

## 技术前提

- DSH 版本：`0.1.2-alpha.1`（升级目标；alpha 版 API 可能继续变化，升级后
  建议在插件里 pin 住 `@deepseek-ai/*` 版本）；
- 插件形态：host/client 双半身 Cordis 插件（`dsh.bundle.patch` + `dsh.client`
  声明）；bundle patch 契约自 `dsh-v0.1.0-rc.7` 起稳定；
- 构建：esbuild（外部插件团队常用）或官方 tsdown preset（仓库内包）。

## 许可

[MIT](LICENSE)
