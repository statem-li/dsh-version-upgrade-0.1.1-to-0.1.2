---
name: dsh-upgrade-host-half
description: DSH 插件升级的 host 半身迁移：0.1.1 升 0.1.2 的宿主侧变化（@Remote/TypertRemoteService 服务面、工具与 ptc 改名、webServer 路由、ctx.inspector）与「@deepseek-ai/* 只发源码不发产物」的解析面大坑（已安装位置 ERR_MODULE_NOT_FOUND）、vendor 化叶子模块、assertHostExternals 构建守卫、已安装位置裸 node 冒烟。触发词：host 半身、ERR_MODULE_NOT_FOUND、@deepseek-ai/dsh-llm 找不到、vendor 化、code-mode ptc、工具注册、webServer.register、host 冒烟。
---

# DSH 插件升级：host 半身

0.1.1 → 0.1.2 的 host 侧变化按频率排序：**解析面（必炸）> 服务面（改了要用
新姿势）> 工具面（改名/扩容）> 路由（谨慎注册）**。绝大部分"升级后启动失败"
都是第一条。

## 1. 解析面：`@deepseek-ai/*` 只发源码，不发运行时产物

**这是最贵的一个坑，先说。**

DSH 的 `@deepseek-ai/*` 包发布时 `lib/` 里只有 `.d.ts` + tsconfig 构建缓存 +
少量手工入口，**没有 `lib/index.js`**——尽管 `package.json` 写着
`"main": "lib/index.js"`。DSH 自己跑在 **tsx** 下，靠 `tsconfig.base.json`
的 `paths` 把包名映射回源码；而 tsx 的 `resolveTsPaths` **只对不在
`node_modules` 里的 importer 应用 `paths`**。插件装进 profile 后住在
`node_modules` 里，拿到的是裸 node 解析 → 必炸：

```
ERR_MODULE_NOT_FOUND: Cannot find module
  '<profile>/node_modules/@deepseek-ai/dsh-llm/lib/index.js'
  imported from ...\node_modules\<plugin>\lib\index.js
```

**实测数据（0.1.2-alpha.1）**：247 个 `@deepseek-ai/*` 包里只有 **82 个**有
`lib/index.js`。源码目录里跑同一份产物完全正常——**构建期成功与运行时可用
完全无关**，这是升级排查最容易误判的地方。

### 判定某个包能否 external（三步，缺一不可）

```bash
# 1. 找到它在 profile 里的位置
d=$(node -e "console.log(require.resolve('@deepseek-ai/X/package.json',{paths:['<profile>/node_modules']}))")
# 2. 有没有运行时入口
ls "$(dirname $d)/lib/index.js" || echo "不可用"
# 3. 有入口还不够：看它自己还 import 什么（dsh-session 有产物但 import dsh-llm，照样炸）
grep -oE "from ['\"][^'\"]+['\"]" "$(dirname $d)/lib/index.js" | sort -u
```

`@deepseek-ai/cordis` 与 `@deepseek-ai/dsh-util-crypto` 是经得起检验的
external 名单——**每次换 DSH 版本都要重验一遍**（0.1.1 → 0.1.2 的包集
变了，白名单不能复用）。

### vendor 化：把需要的叶子模块复制进插件 `vendor/`

叶子模块三条合格标准：

1. **不 import `@deepseek-ai/cordis`** —— 否则拿到第二份 cordis，服务名
   解析与 `instanceof` 全部对不上宿主；
2. 不注册 cordis 服务、不持有跨模块可变状态 —— 纯函数或纯类；
3. 自身依赖要么也在 vendor 内，要么已在安全名单里。

改写 vendor 文件的 import 说明符时：**值导入**（`import { X } from '...'`）
指向 vendor 内本地副本；**类型导入**（`import type { X }`）**保留指向
`@deepseek-ai/*`**——esbuild 会完全擦除，不参与运行时解析。

常见可 vendor 的叶子：`BlockAssembler`/`createUserMessage`/`createMessage`
（`packages/llm/llm/src/{assembler,message}.ts`）、`HarnessError`
（`packages/llm/llm/src/error.ts`，零导入）、`assertNever`、`deepFreeze`、
`isJsonValue`（`packages/core/session/src/json.ts`，190 行零导入）。
注意 `packages/core/tools/src/index.ts`（1945 行）会拉进 cordis、schemastery、
dsh-scope…… **只能取其中类型，别整体内联**。

### 构建守卫（必须加，写完自测）

```js
const HOST_RUNTIME_EXTERNAL_ALLOWLIST = new Set([
  '@deepseek-ai/cordis',
  '@deepseek-ai/dsh-util-crypto',
])

function assertHostExternals(outfile) {
  const source = readFileSync(outfile, 'utf8')
  const specifiers = new Set()
  for (const m of source.matchAll(/(?:^|[;\n])\s*(?:import|export)[\s\S]*?from\s*["']([^"']+)["']/g)) specifiers.add(m[1])
  for (const m of source.matchAll(/\bimport\s*\(\s*["']([^"']+)["']\s*\)/g)) specifiers.add(m[1])

  const violations = [...specifiers].filter(s => !s.startsWith('node:') && !HOST_RUNTIME_EXTERNAL_ALLOWLIST.has(s))
  if (violations.length > 0) throw new Error('unresolvable runtime imports:\n' + violations.join('\n'))
}
```

`import/export from` + 动态 `import()` **双正则一起扫**，漏一种漏判。
**写完故意把某个导入改回 `@deepseek-ai/dsh-llm` 确认构建真的失败**，
没验证过的守卫等于没有。

## 2. 服务面：消费方走 `ctx.remote.*` 命名空间

0.1.2 的 host 服务普遍以 **`@Remote` / `TypertRemoteService`** 模式暴露
浏览器可调的服务面（`LlmRuntime.listProviders()`、
`SubagentRuntime @Remote('interruptByParent')`、
`SettingsController` 的 `settings/*` 方法等）。插件 host 半身消费这些能力
时的注意点：

- **行级 inject 声明**：需要哪个服务就 inject 哪个，**声明不一致 =
  不挂载或 `without inject`**。实测红线：
  - `directoryFor` 需要 `remote` + `remote.session` **双注入**；
  - 操作 settings schema 必须行级 inject 声明 `settingsSchema`。
- **`dsh-client-schema-form` 不在 client 模块表**（0.1.2 本机实测）：
  需要 schema 表单的操作一律走 `ctx.settingsSchema`，剔除对 schema-form 的
  运行时 import——否则 client 半身 `require` 即 `missed the module table`。
- **`ctx.inspector`**（0.1.2 新增，experimental）：发布一个 JSON 观察值，
  供宿主/客户端检查面读取。想增强可诊断性的插件可用，不是必须。
- **插件自有设置**：0.1.2 是插件自有 settings 命名空间（
  `plugin-owned settings surface`），**升级时命名空间名沿用旧值**——
  改名 = 用户配置全丢。老配置格式变化时做一个「读旧 → 写新」的一次性
  迁移函数，而不是静默丢弃。

## 3. 工具面：`code-mode` → `ptc` 改名与工具 catalog 扩容

0.1.2 把工具执行模式从 `code-mode` 概念改名为 **`ptc`**
（`packages/core/tools/src/code-mode.ts` → `ptc.ts`）。升级自查：

- 插件工具的参数 schema / 结果解析里出现 `codeMode`、`code-mode` 字样 →
  改为 `ptc` 相关字段；
- 接 `tools/ptc-dispatch-log` 这类事件线的插件注意事件名变化；
- `@deepseek-ai/dsh-tools` 的 `defineTool` 用法以
  `packages/core/tools/src/index.ts`（0.1.2 版）为权威——**用 grep 对
  catalog 而不是凭记忆**：`packages/extensions/tool-cordis/src/api-catalog.ts`
  是模型可见工具的目录，0.1.2 相比 0.1.1 大改（+1387 行）。

## 4. HTTP 路由：前缀独立 + 同 path 报警

- 插件路由挂 `ctx.webServer.register({ kind: 'exact'|'prefix', path, handler })`；
  **重复 path 会 warn 并跳过注册**（不报错、不保证第一个赢）——升级时
  检查所有路由前缀是否与其它已装插件重复，统一加插件前缀
  （`/api/<plugin-id>/...`）。
- 0.1.2 的 web 组合包用 `registerFallback` / `applyIndexTaps` 拥有静态
  dist——第三方插件**不要**碰 fallback 席位（那是 SPA 服务器的单一所有者）。

## 5. 冒烟测试（升级验收的强制项）

- **宿主产物自包含时裸 node 即可跑**（不需要 tsx）；栈桩 ctx 至少给
  `logger / webServer.register / tools.register / on / get / effect`，
  路由用 Map 收集以检测重复 path。
- **真正的验收在「已安装位置」再跑一遍**（不加 tsx）：
  `cd <profile>/node_modules/<plugin> && node scripts/smoke-host.mjs`。
  源码目录通过而这里失败 = 第 1 节解析坑。
- 每加一个功能模块就补一条对应路由/工具的断言——**漏模块这种 bug 只能靠
  断言防回归**。
- Windows 下从 Git Bash 传路径给 node 会被转换坏（`/d/...` → `D:\d\...`），
  用 Windows 风格路径。

## 6. 多模块挂载的铁律

插件往往塞了三四个独立功能：**每个模块各自 try/catch 挂载**，一个挂了
不能拖垮兄弟模块——不要整个 `apply` 包一层 try/catch 了事。

## 相关

- 主路线：`dsh-upgrade-plugin-011-to-012`
- 清单与构建产物：`dsh-upgrade-manifest-contract`
- client 侧 Remote 消费迁移：`dsh-upgrade-client-half`
