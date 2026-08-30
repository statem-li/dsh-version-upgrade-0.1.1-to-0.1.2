---
name: dsh-upgrade-plugin-011-to-012
description: 把已有的 DeepSeek Harness 第三方插件 repo 从 DSH 0.1.1（dsh-v0.1.1-rc.1/rc.2）升级到 0.1.2-alpha.1 的迁移总览与路线。面向插件作者，不是插件用户。覆盖：升级前的依赖面盘点、manifest 契约对齐、host/client 半身迁移、构建分发与已安装位置验收；含「症状 → 根因 → 修法」速查表和官方文档地图。触发词：插件升级、0.1.1 升 0.1.2、插件 repo 迁移、插件打包契约、bundle patch、插件加载失败、插件升级后入口消失、插件升级后 404、插件升级后启动失败。子技能：dsh-upgrade-manifest-contract、dsh-upgrade-host-half、dsh-upgrade-client-half。
---

# DSH 插件升级：0.1.1 → 0.1.2

**这个技能干什么**：把已有插件 **repo 里的代码与清单**从 DSH 0.1.1 升级到
0.1.2。这是**增量迁移**，不是从零开发——从零写插件看官方
`docs/cordis-tutorial/` 与 `docs/cookbook/`，本技能不重复。

**版本背景**：`dsh-v0.1.1-rc.1/rc.2`（2026-08-21）→ `dsh-v0.1.2-alpha.1`
（2026-08-28），区间约 1100+ commits，是一次大重构（web 客户端重写、服务面
Remote 化、核心重命名、新增 bundle 形态）。0.1.2 仍是 **alpha**：API 可能继续
变化，升级后建议把 `@deepseek-ai/*` 版本 **pin 住**再让用户安装。

## 迁移五步路线

### 第 1 步：盘点（一天内完成，别跳过）

升级前先回答三个问题，答案决定后面全部工作量：

1. **依赖面**：插件 import/require 了哪些 `@deepseek-ai/*`？是否 runtime 依赖？
   用 `require.resolve('@deepseek-ai/X/package.json', { paths: [profile 的
   node_modules] })` 实测它在**已安装位置**有没有运行时入口。
2. **服务面**：用了哪些 host 服务（`ctx.webServer` / `ctx.tools` /
   `ctx.settingsSchema` / `ctx.remote.*` / `ctx.slots`…）与 `cordis/*` 事件？
3. **槽位面**：client 注册了哪些槽位？是否覆盖了内核占用的座位？

> 做法建议：建一张「依赖 → 用途 → 升级动作」表，逐行过。漏一项就是上线后一个
> 静默 404 或无声的入口消失。

### 第 2 步：manifest 与构建契约（→ `dsh-upgrade-manifest-contract`）

- `package.json` 五件套：`main` / `exports`（含 `./client`、`./cordis.patch.yml`、
  `./package.json`）/ `files` / `dsh.bundle.patch` / `dsh.client`。
- 老 repo 若靠**手写 `cordis.patch.yml` insert 行**挂载：迁移到 bundle patch
  自动注册，并清理遗留 insert（三个坑：包未安装→启动失败、重复注册、
  last-write-wins 合并）。
- 构建产物：双 bundle（host ESM 自包含 + client CJS factory），产物随仓库提交，
  **不要 `prepare` 脚本**（pnpm ≥ 10 拦截 git 依赖构建）。

### 第 3 步：host 半身（→ `dsh-upgrade-host-half`）

- 工具注册与 schema（0.1.2 的工具面扩容；注意 **`code-mode` → `ptc` 改名**）。
- 服务消费：`ctx.remote.*` 命名空间（`@Remote` / `TypertRemoteService` 模式）。
- `@deepseek-ai/*` **解析面**：0.1.2 的这些包只发源码不发产物（实测 247 个包
  仅 82 个有 `lib/index.js`），已安装位置裸解析必炸——按白名单验证 + vendor 化。
- 构建守卫 `assertHostExternals()`：没有它，坏掉的插件只在用户机器上启动失败。

### 第 4 步：client 半身（→ `dsh-upgrade-client-half`）

- **槽位系统改为声明制**：`ctx.slots.register({ name, id?, order?, children?,
  store?, inject?, locale? }, Component)`，四种 cardinality + 三种 scope，
  冲突即激活失败。
- **旧 unary API 调用迁移到 Remote 命名空间**（`session.rename` →
  `ctx.remote.sessionTitle.rename` 等，迁移对照表见子技能）。
- **注入模块表**：`dsh.client.inject` 只列模块表运行时行，不在表内的包
  require 即启动失败（`missed the module table`）。
- Web 客户端这版重构了 DOM（lexical 输入框、会话树、turn-tail 操作区等）：
  插件挂载点全部 **grep 实证**，注释里的宿主 id 会过期。

### 第 5 步：验收（发布前，在「已安装位置」跑）

1. `--dump-config` 能看到 `patched by <plugin> + - id: <plugin>`（bundle patch 生效）；
2. profile 的 `dsh.profile.bundles` 与 `dependencies` 都写上了；
3. **已安装位置**裸 node 跑宿主冒烟（源码目录通过不算数）；
4. 产物与本地构建一致（忽略换行差异）。
5. 逐项功能回归：入口可点、路由 200、槽位不冲突、**用户配置与数据保留**
   （localStorage 键 / settings 命名空间 / 已读记录等）。
6. **提醒用户重启一次 DSH**（宿主半身不支持热重载；client 端产物刷新即生效）。

## 变化速查表（症状 → 根因 → 修法）

| 症状 | 根因 | 修法 |
| --- | --- | --- |
| 装完启动失败：`ERR_MODULE_NOT_FOUND: Cannot find module '@deepseek-ai/dsh-llm/lib/index.js'` | host 产物 runtime import 了 `@deepseek-ai/*`，已安装位置解析不到（只发源码） | 白名单验证 + vendor 化 + `assertHostExternals` 守卫（`dsh-upgrade-host-half`） |
| `missed the module table` / 启动即挂 | `dsh.client.inject` 里列了模块表中不存在的包（如 `dsh-client-schema-form`） | 只列模块表运行时行；schema 走 `ctx.settingsSchema`（`dsh-upgrade-client-half`） |
| 侧边栏入口只有一部分 / 全不显示 | 宿主 DOM 变化（旧 host id 已被移除，注释骗人） | grep 实证宿主 id；槽位由插件自己的 ensure 逻辑创建（`dsh-upgrade-client-half`） |
| `already registered` 抛异常（非警告） | 与内核或另一插件占同座位（`('/', 'skill')`、keyed 槽同名） | 插件 bundle patch 里停用内核插件；新插件槽位 id 带前缀（`dsh-upgrade-client-half`） |
| UI 出来了但 `/api/*` 一片 404 | 一个 UI 功能对应多个 host 模块，漏迁 host 侧 | 列出客户端所有调用路径逐条 grep 是否注册（`dsh-upgrade-host-half`） |
| 升级后原配置/已读位置丢失 | 改了 localStorage 键 / CSS 类名 / settings 命名空间 | 拆分/融合时契约延续：键、类名、命名空间一律沿用旧值（`dsh-upgrade-client-half`） |
| `err_pnpm_git_dep_prepare_not_allowed` | 包里有 `prepare`/`postinstall`（pnpm ≥ 10 拦截） | 删脚本，提交 `lib/`（`dsh-upgrade-manifest-contract`） |
| 安装后副作用没生效，页面刷新也没用 | 改的是 host 半身（不支持热重载） | 重启 DSH（`dsh-upgrade-manifest-contract`） |
| 弹层/全屏遮罩塌进卡片 | `position: fixed` 的祖先带 `backdrop-filter`/`transform` | `createPortal(…, document.body)`（`dsh-upgrade-client-half`） |
| smoke 在源码目录过、已安装位置失败 | 解析坑（构建期成功 ≠ 运行时可用） | 已安装位置裸 node 重跑（`dsh-upgrade-host-half`） |

## 官方文档地图（按需取用，本技能不重复其内容）

- 插件子系统现状：`docs/subsystems/extensions.md`
- 槽位系统：`docs/subsystems/slots.md`；web 客户端：`docs/subsystems/web-client.md`
- client 包规则：`packages/client/AGENTS.md`
- 开发教学：`docs/cordis-tutorial/`、`docs/cookbook/`
- 架构笔记（迁移真相所在）：`.agents/notes/implemented/architecture/`
  - `2026-08-05-profile-plugin-bundles.md`（profile/bundle 机制）
  - `2026-08-10-unary-apiproxy-remote-migration.md`（unary API → Remote 迁移表）
  - `2026-07-23-client-plugin-loading-model.md`（client 装载模型）
  - `2026-08-15-client-shells-and-dynamic-packages.md`（client 包分类与构建面）
  - `2026-08-12-plugin-owned-settings-surface.md`（插件自有设置面）

## 完成定义（DoD）

- [ ] 依赖面盘点表逐行执行完毕；
- [ ] 老 repo 无遗留手工 insert；`dsh plugin add` 一次装成功；
- [ ] host 产物裸 node 可在已安装位置加载；
- [ ] client 槽位注册全部成功、无 `already registered`；
- [ ] 升级前用户可见的配置/数据在新版上原样保留；
- [ ] 发布前在干净 profile 上完整验收过一遍。
