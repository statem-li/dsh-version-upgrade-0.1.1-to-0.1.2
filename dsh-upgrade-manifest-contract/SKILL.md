---
name: dsh-upgrade-manifest-contract
description: DSH 插件升级的清单与构建契约面：package.json 五件套（main/exports/files/dsh.bundle.patch/dsh.client）、bundle patch 自动注册替代手工 insert 的清理规则、双 bundle 构建（host ESM 自包含 + client CJS factory）、产物随仓库提交且不带 prepare 脚本、files 精简与换行符、安装失败的三种常见原因、已安装位置验收四步。触发词：package.json 契约、bundle patch、dsh.client、exports、files、prepare 拦截、plugin add 失败、dump-config 看不到插件。
---

# DSH 插件升级：清单与构建契约

升级的第一步是把插件的**分发清单**对齐到 0.1.2 的现行契约。0.1.1 与 0.1.2
的 bundle patch 契约写法一致（`dsh.bundle.patch` 自 `dsh-v0.1.0-rc.7` 起），
真正的升级工作在于：**老 repo 往往没写过、或还在用手工 insert、或用旧式
单 bundle**。

## 1. package.json 五件套

```json
{
  "name": "<plugin-id>",
  "type": "module",
  "main": "./lib/index.js",
  "exports": {
    ".": "./lib/index.js",
    "./client": "./lib/client.js",
    "./cordis.patch.yml": "./cordis.patch.yml",
    "./package.json": "./package.json"
  },
  "files": ["lib", "scripts", "build.mjs", "cordis.patch.yml", "README.md"],
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },
    "client": {
      "platform": "web",
      "inject": ["@deepseek-ai/dsh-client-ui-slots", "..."]
    }
  }
}
```

逐项检查点：

- **`exports["./client"]`**：client 半身必须通过子路径暴露，host 与 client
  用同一包名两个入口——漏了 `./client` 等于 client 半身永远加载不到。
- **`exports["./cordis.patch.yml"]`**：bundle patch 机制按此子路径读取插件
  自带的 patch；漏了会静默退化为不注册。
- **`dsh.client.platform: "web"`** 与 **`dsh.client.inject`**：inject 里
  每个包名必须是所在 DSH 版本 **client 模块表的运行时行**（不是 npm 包列表！）。
  多列、少列都出问题：多列 → `missed the module table` 启动即挂；少列 →
  client 半身需要的服务拿不到。验证方法见
  `dsh-upgrade-client-half`。
- **`peerDependencies` 范围写法**：`"@deepseek-ai/cordis": ">=4.0.0-rc <5"`、
  `"@deepseek-ai/dsh-agent": ">=0.0.1-rc <2"` 这类开放区间是官方惯例；
  **升级到 0.1.2 后建议收紧**（或至少验证当前实际版本在范围内）。

## 2. 从手工 insert 迁移到 bundle patch 自动注册

老 repo（0.1.1 时代常见）在用户侧要求往
`<profile>/cordis.patch.yml` 手写：

```yaml
- insert:
    - id: <plugin-id>
      name: <plugin-id>
```

**迁移动作**：改为插件自带 patch 后，删除该手工条目。三个真实坑：

1. **「包未安装 → 启动失败」**：insert 行的包若没进 profile 的
   `dependencies`，启动时挂载直接失败；
2. **「bundle patch 重复注册」**：插件自带 patch + 手工 insert 两边都写，
   reconcile 不知道听谁的；
3. **改覆盖用 last-write-wins**：profile 自己的 patch（更晚的层）覆盖插件
   patch——插件作者写在自带 patch 里的东西（比如停用内核插件）可以被用户
   覆盖，文档里要说明这一点。

### 插件自带 patch 的内容

```yaml
# 由 `dsh plugin add` 自动并入 profile；不要叫用户手工编辑。
- id: ui-skill
  name: '@deepseek-ai/dsh-client-ui-skill'
  disabled: true      # 与内核抢座位时才需要，见 dsh-upgrade-client-half

- insert:
    - id: <plugin-id>
      name: <plugin-id>
```

**卸载即撤销**：`dsh plugin remove` 后 patch 整体消失，被它停用的内核插件
自动恢复——文档里写清楚「卸载本插件后内核 `/` 菜单会回来」，别让用户体验
"卸载了插件为什么 ui-skill 消失了"。

### 老插件停用条目

已卸载/已停用插件在**运行中的 fiber 里还活着**。profile 的 patch 里保留
`- id: <old-plugin>/name: <old-plugin>/disabled: true` 供热停用，**下次重启后
即可删除**。告诉用户这个顺序，避免"我明明卸载了还在"的困惑。

## 3. 双 bundle 构建

一个构建产出两个文件：

| 产物 | 形态 | 平台 | 规则 |
| --- | --- | --- | --- |
| `lib/index.js` | ESM | node | **自包含**：除 `node:` 外只留逐个验证过白名单（见 `dsh-upgrade-host-half`） |
| `lib/client.js` | CJS | browser | `window.__ModuleLoader__.load({ id, factory })` 工厂外包；`react`/`react-dom`/`react/jsx-runtime` 与 `@deepseek-ai/*` 全部 external |

client 工厂骨架（esbuild）——**注意 esbuild 没有 `intro`（那是 Rollup 的），
CJS 前导只能 `banner`/`footer` 拼**：

```js
banner: { js: [
  `window.__ModuleLoader__.load({ id: ${JSON.stringify(PLUGIN_ID)}, factory: (require) => {`,
  'var module = { exports: {} };',
  'var exports = module.exports;',
].join('\n') },
footer: { js: 'return module.exports; } });' },
```

host 半身若内联了 CJS 依赖（如 yauzl），ESM 产物会报
`Dynamic require of "fs" is not supported`——加 banner：

```js
"import { createRequire as __cR } from 'node:module';",
"const require = __cR(import.meta.url);"
```

**仓库内官方包**用共享 tsdown 预设（`packages/client/tsdown.client.ts`），
**外部插件 repo 独立构建**（esbuild 即可）。两条路都要求：产物可手动重跑、
产出 `client.js.map`（client 侧 sourcemap 链是正式能力，combo 加载按 map
回溯源码）。

## 4. 产物随仓库提交，不要 prepare

pnpm ≥ 10 默认拦截 git 依赖的构建脚本：

```
ERR_PNPM_GIT_DEP_PREPARE_NOT_ALLOWED
```

**正解：根本不写 `prepare`**，把 `lib/` 提交进仓库：

- `.gitignore` **不要**忽略 `lib/`（写清楚原因，否则下一个人会"顺手"加回去）；
- `package.json` **不要**有 `prepare` / `postinstall`；
- `files` 保留安装后还需要的东西——**漏了 `scripts/` 用户跑不了冒烟自测**。

报错末尾那句 `"git-hosted plugins build on install via their prepare script…"`
是通用兜底提示，**不要轻信**——没有 prepare 的包也会打这句话。真正的失败
原因在它上方的 `ERR_*` 行。

## 5. 换行符（Windows 老坑）

Windows 检出把 LF 转 CRLF，安装副本与本地产物 md5 对不上，排查极耗时。
`.gitattributes`：

```
lib/*.js   text eol=lf
lib/*.map  -text
*          text=auto
```

跑一次 `git add --renormalize lib/`。

## 6. files 只发运行时

`src/`、`vendor/` 别发出去（esbuild 已内联）。实测把 93 个文件的包砍到
10 个，`dsh plugin add` 从 2m5s 降到 7.4s——git 依赖安装会先把整包复制进
临时目录再删，文件数直接决定开销。

## 7. 安装失败的三种常见原因

1. **`ERR_PNPM_GIT_DEP_PREPARE_NOT_ALLOWED`** —— 真的有 `prepare`（见上）；
2. **`[safe-delete][SAFE_DELETE_BULK_CONFIRM_REQUIRED]`** —— pnpm 一轮内
   累计删除数超阈值，计数是 **turn 级累计**：`remove` 和 `add` 必须分两轮
   执行，写进同一条命令必然失败；
3. **`atomic-write: timed out waiting for the writer lock`** —— 上次中断
   留下死锁。锁文件里是持有者 PID，确认该 PID 已不存在再删：
   `.dsh/profiles/node_modules.lock`。

失败 add 留下的残留 `<plugin-id>_tmp_<pid>_<n>/` 目录无害，确认没自己的
改动后可删。

## 8. 已安装位置验收（发布前必做）

```bash
# 1. bundle patch 生效：应看到 patched by <plugin> +  - id: <plugin>
node --import tsx/esm apps/cli/lib/bin.js --profile web --dump-config | grep -B2 -A3 '<plugin-id>'

# 2. profile 的 bundles 与 dependencies 都写上了
node -e "const p=require('<profile>/package.json');console.log(p.dsh.profile.bundles, p.dependencies)"

# 3. 在【已安装位置】用【裸 node】跑宿主冒烟（关键！源码目录通过不算数）
cd <profile>/node_modules/<plugin-id>
node scripts/smoke-host.mjs
```

宿主半身不支持热重载：**装完一定提醒用户重启一次 DSH**；client 半身产物
由宿主按 no-cache 从磁盘读取，普通刷新页面即生效。

## 相关

- 主路线：`dsh-upgrade-plugin-011-to-012`
- host 产物解析面与冒烟：`dsh-upgrade-host-half`
- client 注入与模块表：`dsh-upgrade-client-half`
