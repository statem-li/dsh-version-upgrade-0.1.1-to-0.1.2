---
name: dsh-upgrade-client-half
description: DSH 插件升级的 client 半身迁移：0.1.1 升 0.1.2 的浏览器侧变化——槽位声明制（ctx.slots.register 对象形态、cardinality/scope/priority、组件四 shares props、hooks 五座）、旧 unary API 到 Remote 命名空间的迁移对照表（session.rename→sessionTitle/rename 等）、dsh.client.inject 模块表真相（不在表内即 missed the module table）、web 重构对挂载点的影响（grep 实证、MutationObserver 兜底、createPortal 到 body、ErrorBoundary 包面板）。触发词：client 半身、slot 注册、already registered、槽位不显示、Remote 调用、missed the module table、createPortal、弹层塌陷、0.1.1 升 0.1.2 界面不显示。
---

# DSH 插件升级：client 半身

0.1.2 的浏览器侧是一次大重构：槽位系统改为**声明制**、旧 unary API 调用
全部迁入 **Remote 命名空间**、会话数据只经 **hooks 快照渠道**。这一步的坑
特征是「**不报错**」：入口静默消失、槽位没渲染、某块数据空白——所以要按
下面的清单逐项改，而不是等报错。

## 1. 槽位系统：0.1.2 声明制

### 注册 API

```tsx
// 0.1.2 声明制：对象形态 + Component
ctx.slots.register({
  name: 'conversation.session.header.actions',   // <domain>.<entry>.<hole>
  id: 'review',          // list 槽位的必填 id
  order: 100,            // list 槽位的顺序；无则注册顺序
  // priority: 影子替换排名（single/list/keyed 的替换候选，低值优先）
  // children: 本组件渲染的子槽位声明（{ [key]: { kind, scope } }）
  // store: 共享视图状态的 store factory
  // inject: 私有数据/回调工厂（apply 闭包 ctx 的注入面）
  // locale: 本地化命名空间
}, HeaderAction)
```

四种 cardinality：`single`（一个格子，影子替换）/ `list`（按 `id` 寻址、
`order` 排序）/ `keyed`（owner 按 `entryKey` 派发）/ `chain`（每个 entry 提供
`select(owner)`，首个非空胜出）。三种 scope：`root` / `session-maybe` /
`session`。

**升级自查**：

- 0.1.1 旧式 `ctx.slots.register('shell.overlay', Comp)` / 手写 face 的写法
  全部改成对象形态；**注册进未声明的槽位、或声明了别人已拥有的 child，
  都是激活失败**（不是警告）——"冲突是设计在说话"，不要绕过。
- 贡献**其他包**的槽位用 `ctx.slots.inject(name, callback)`：回调用
  `ctx.slots.register({...})` 完成登记，随 owner 生命周期收放。
- 与其抢占内核/别的插件已占的 `single`/`keyed` 格子，不如选一个**新的
  list id 或 keyed key**；确实要替换内核 UI 时用 `priority` 并写进
  bundle patch 停用内核插件（`ui-skill` 占 `('/', 'skill')` 与
  `tool.call.toolview` 的 key `skill` 两个座位）——重复注册直接抛
  `already registered`。

### 组件输入：四 shares（+ locale），业务组件永远拿不到 ctx

| 输入 | 类型 | 来源 |
| --- | --- | --- |
| owner 与标准 scope 值 | `PropsRuntime<K>` | SlotMap 行 + scope 适配器 |
| 授权的子槽位渲染器 | `PropsRenderSlots<S>` | register 的 `children` |
| 共享视图状态的 selector/actions | `PropsStore<H>` | register 的 `store` |
| 私有数据、回调、可观察 hooks | `InjectFace<I>` | register 的 `inject` |
| 本地化 `t` 函数 | `PropsLocale<N>` | register 的 `locale` |

**hooks 五座**：`useSession`、`useSessions`、`useWorkspaces`、`useStore`、
`renderSlot`，外加渲染器从 provide 贡献绑定的 `use<Name>`。规则：

- 组件里**禁止**手写 hook 作为 prop、禁止 import 服务类去戳、禁止读 React
  context、禁止碰 ctx；
- 渲染代码读可变数据**必须**经框架 hook；事件处理代码可以读快照；
- 共享/可重挂数据用 store（`createXXXStore()` 工厂，禁止模块级单例）；
  跨包行为走注入服务，跨包 UI 走槽位——**永远不要 import 另一个功能插件的
  运行时代码**。

### 0.1.1 → 0.1.2 数据渠道变化

- 快照 selector 体系重排：`useSession` 与 `useConversation` 分离
  （`useSession` 现在给 `SessionSnapshot`）、新增 `useTrajectory` /
  `useProjection` / `useSessionPendingInteraction` / `useInput` 等；
- 会话历史传输 0.1.2 走 packed 传输 + webworker 化——插件**别再自己
  fetch 会话数据**，一律从 `useProjection` / `useConversation` 等渠道取。

## 2. 旧 unary API → Remote 命名空间迁移（对照表）

0.1.1 时代 client 经 API Proxy 一元调用的一批操作，0.1.2 全部迁入所属
业务 Remote 命名空间（`.agents/notes/implemented/architecture/2026-08-10-unary-apiproxy-remote-migration.md`）。
**插件里凡出现这些调用必须改**，否则升级后直接不可用：

| 旧（API Proxy 一元） | 新（Remote 命名空间） |
| --- | --- |
| `session.rename` | `sessionTitle/rename` |
| `command.list` / `command.execute` | `commands/list` / `commands/execute` |
| `llm.providers` | `llm/listProviders` / `llm/listConfigurableProviders` |
| `llm.discoverModels` | `llm/discoverModels` |
| `llm.models` | `session/modelCatalog` |
| `credentials.describe/set/unset` | `credentials/describe/set/unset` |
| `settings.describe/update/replace/mutate` | `settings/*` 同名方法 |
| `settings.openDocument` | `settings/openSettingsDocument` |
| `agentPreset.read/copy/remove` | `agentPresets/*` |
| `agentPreset.openDocument` | `settings/openAgentPresetDirectory` |
| `subagent.interrupt` | `subagents/interruptByParent` |
| `workspace.list/insertSessionBefore/archiveSession` | `workspace/*` |
| `skill.list` | `skills/list` |
| `fileReferences/list` | `fileReferences/list` |
| `host.openPath` | `session/openWorkspacePath` |
| `host.describe` | `$events` ready frame + 各 owner 的 capability 查询 |
| `session.export` | `GET`/`HEAD /api/session.export`（精确 Fetch 路由） |

保留行为（迁移时不要"顺便简化"）：Agent 绑定的调用必须走共享 lookup 策略
（live 复用、冷恢复、并发去重、subagent ownership fence）；`skill.list`
**不激活 Session**；`llm/*` 保留 provider 失败净化与取消。
**Remote 签名或所选包变化时必须重新生成 Remote 产物与 API Remotes assembly**。

## 3. `dsh.client.inject`：模块表真相

- client 半身走浏览器**模块表**，由 DSH 提供同一份实例——不存在宿主侧
  那种 node 解析问题，`@deepseek-ai/dsh-client-*` 可以放心 external；
- **但**：inject 里每个名字必须是所在 DSH 版本模块表的**运行时行**。
  0.1.2 实测 `dsh-client-schema-form` **不在模块表**——`require` 即
  `missed the module table` 启动失败；
- 有的包名能列出但运行时行不存在（「上一版能用 ≠ 这版能用」）；
  验证方法：对 module 表 catalog 逐个核；
- `dsh.client.external` 是**基础设施专用**（传输/生成 assembly 的模块表
  物化请求），**不是功能插件互相拿代码的机制**——需要共享代码走注入服务
  或槽位；
- 可选 `dsh.client.immediately` 标记让它随启动预取。

## 4. Web 重构对挂载点的影响

0.1.2 的 web 客户端重写了输入框（Lexical）、会话树、turn-tail 操作区等。
插件升级时：

- **任何「由 XXX 渲染/提供」的说法都要 grep 实证**：旧版 webui 注释说
  「skills/memory 槽位由 AutomationApp 渲染在 `#dsh-automation-menu-host`
  内部」——该 host id 在 0.1.2 里根本不存在，注释不报错，只会静默不显示；
- 槽位宿主节点由插件自己的 `ensureHostPlaced()` 创建，统一插到目标 slot
  之前；用一个显式行布局表表达「哪些槽位并排」；
- 宿主是插进 React 树的裸节点，children 重排可能回收它：轮询兜底 +
  `MutationObserver` 盯**直接父节点**（`childList`，**不要开 subtree**），
  补位后就位检测为真、不再改 DOM，避免自激；
- **React 18 渲染期异常默认卸载整个 `createRoot` 树**：面板里一个组件
  抛错，入口按钮跟着一起没，且控制台无痕。修法两条：
  1. 弹层一律 `createPortal(…, document.body)` —— 顺带解决
     `position: fixed` 被 `backdrop-filter`/`transform` 祖先钉进局部
     坐标系（全屏遮罩塌进卡片的那类 bug）；
  2. 每个面板外面、入口按钮里面包 **ErrorBoundary**（`label` + `fallback`
     + `onError`），崩了只收面板、留按钮。

## 5. 插件间并存与数据契约（拆分/融合场景）

- **路由与槽位前缀独立**：新插件用私有路径 `/api/<plugin-id>/...`、槽位 id
  带插件前缀，与其它插件（含被拆分的全家桶）并存不冲突；
- **数据契约延续**：localStorage 键、CSS 类名、样式表 id、settings
  命名空间**一律沿用旧值**——旧配置（位置/已读/开关/外观/模型能力开关）
  无缝延续；改名 = 用户配置全丢；
- **删掉依赖其它模块的入口**：拆出单体时 grep 一遍自定义事件桥/跨模块
  dispatchEvent，无监听者的入口直接移除（不是留着）；
- **行为契约原样保留**：原先的边界语义（subagent 跳过、aborted 不算、
  空回合不上报、每页加载先全量恢复再增量……）写进 README「设计约束」，
  别顺手"优化"掉。

## 相关

- 主路线：`dsh-upgrade-plugin-011-to-012`
- host 半身与解析面：`dsh-upgrade-host-half`
- 清单与构建：`dsh-upgrade-manifest-contract`
- 官方槽位权威文档：`docs/subsystems/slots.md`、`packages/client/AGENTS.md`
