# Windows 开发适配记录

在 Windows 上跑通 `pnpm tauri dev`（`chat-agent` profile 的 desktop 开发）时，暴露并修复了若干配置缺失与链路问题。本文记录改动及其原因，供后续在 Windows 上开发、或向其他平台迁移时参考。

## 背景

`apps/desktop` 是 Tauri 桌面客户端，启动时以 `--profile chat-agent` 拉起 dsh 宿主进程。`chat-agent` 是一个 in-repo 的 profile（`.dsh-home/profiles/chat-agent/`，被 `.gitignore` 忽略的本地开发目录），它在 `dsh-base` bundle 之上叠加 chat 插件族（bot registry、group orchestration、复合搜索 provider）。

本次在 Windows 上依次暴露了三个层面的问题：构建、插件包解析、profile 插件激活。下面按问题分述。

## 1. 构建：`chat-web-search` 缺失 project reference

### 现象

根目录执行 `pnpm run build` 报错：

```
ERROR  Error: Build failed with 1 error:
[UNRESOLVED_ENTRY] Cannot resolve entry module lib/types/index.js.
```

### 原因

`packages/chat/chat-web-search` 是新加入的 host 面包，有自己的 `tsdown.config.ts`（入口 `lib/types/index.js`），但**没有被任何 aggregate（`tsconfig.host.json`）引用**。

构建流程是 `tsc -b tsconfig.host.json`（产出各包 `lib/types/*.js`）→ `tsdown`（rolldown 打包 `lib/types/index.js` → `lib/index.js`）。由于该包不在 references 里，`tsc -b` 从不编译它，`lib/types/index.js` 从不生成；而 tsdown 的 workspace 扫描（`packages/*/*`）仍把它纳入构建，rolldown 找不到入口文件，报 `UNRESOLVED_ENTRY`。

同组的 `chat-bots`、`chat-group`、`chat-agent` 都已在 references 中，唯独这个新增包被遗漏。

### 修复

在 `tsconfig.host.json` 的 `references` 中补上：

```json
{ "path": "./packages/chat/chat-agent" },
{ "path": "./packages/chat/chat-web-search" },
{ "path": "./packages/boot/app-boot" },
```

## 2. tsx source launch：`chat` 组缺失 source 路径映射

### 现象

运行 `pnpm run verify-cordis-config` 报错：

```
packages\chat\chat-agent\cordis.patch.yml: @deepseek-ai/dsh-chat-web-search does not resolve to workspace source through tsconfig.base.json paths
```

（`chat-bots`、`chat-group` 也报同样的错误。）

### 原因

`tsconfig.base.json` 的 `paths` 里，`@deepseek-ai/dsh-*` 的通配符列表没有包含 `./packages/chat/*/src`。`packages/chat/` 组是新加的，但 `tsconfig.base.json` 的 source 路径映射没有同步更新。

`tsx` 的 source launch（`node --import tsx/esm`）依赖这个 paths 映射把包名解析到 workspace source，否则会退回到依赖已构建的 `lib/`。`verify-cordis-config` 强制要求 cordis.yml 引用的 bare plugin 能通过 paths 解析到 source。

### 修复

在 `tsconfig.base.json` 的 `@deepseek-ai/dsh-*` 通配符里补上：

```json
"./packages/code-runtime/*/src",
"./packages/chat/*/src",
"./packages/fs/*/src",
```

## 3. bundle 依赖声明：`chat-agent` 缺 `dsh-chat-web-search`

### 现象

`pnpm tauri dev` 启动时报：

```
Error: dsh: plugin tree failed to load: failed to apply loader entry include (cordis:include):
failed to import loader entry chat-web-search (@deepseek-ai/dsh-chat-web-search):
Cannot find package '@deepseek-ai/dsh-chat-web-search' imported from D:\...\.dsh-home\profiles\chat-agent\
```

### 原因

`chat-agent` 的 `cordis.patch.yml` 通过 `insert` 引用了 `chat-web-search`（loader entry），但 `packages/chat/chat-agent/package.json` 的 `dependencies` 只声明了 `chat-bots`、`chat-group`，漏了 `chat-web-search`。

项目约定（`verify-cordis-config` 强制）：cordis.yml 引用的 bare plugin 必须出现在其 resolver manifest 的 `dependencies` 里。缺依赖导致 pnpm 未为该包建立 workspace 链接，运行时 import 失败。

### 修复

在 `packages/chat/chat-agent/package.json` 的 `dependencies` 补上：

```json
"@deepseek-ai/dsh-chat-web-search": "workspace:^",
```

## 4. profile 解析闭包：`chat-agent` profile 缺依赖与 base bundle

这一层暴露了 profile 的两个独立缺失。

### 4.1 缺 `dsh-chat-web-search` 依赖

`.dsh-home/profiles/chat-agent/package.json` 的 `dependencies` 只声明了 `chat-agent`、`chat-bots`、`chat-group`，漏了 `chat-web-search`。pnpm 只为直接声明的 workspace 依赖建立 `node_modules` 链接，导致 `chat-web-search` 无法从 profile 目录解析。

### 4.2 缺 `dsh-base` bundle（关键）

`chat-agent` 是 "desktop-surface layer **over dsh-base**"：它的 patch 只做增量叠加（patch `system-prompt`、`web`、`tools` 等 id，insert chat 插件），而这些 id 及它们的服务 provider（`dsh-session`、`dsh-llm`、`dsh-agent`、`dsh-tools`、`dsh-web`、`dsh-session-persistence-jsonl` 等）全部定义在 `dsh-base` bundle 里。

但 profile 的 `dsh.profile.bundles` 只列了 `@deepseek-ai/dsh-chat-agent`，漏了 `@deepseek-ai/dsh-base`。结果 `dsh-base` 的所有基础 provider 都没挂载，chat 插件全部因缺服务而 pending：

```
@deepseek-ai/dsh-workspace: pending (waiting for service: sessionPersistence)
@deepseek-ai/dsh-host-apiproxy: pending (waiting for services: agentDefaultModel, agents, ...)
@deepseek-ai/dsh-chat-bots: pending (waiting for services: agents, tools, skills, attachments)
@deepseek-ai/dsh-chat-web-search: pending (waiting for service: web)
@deepseek-ai/dsh-chat-group: pending (waiting for services: chatBots, agents)
```

标准 profile 模板（`PROFILE_TEMPLATES`）都以 `dsh-base` 为第一层，例如 `web: ['@deepseek-ai/dsh-base', '@deepseek-ai/dsh-web-app']`。`chat-agent` profile 遗漏了这一点。

### 修复

在 `.dsh-home/profiles/chat-agent/package.json` 补上 `dsh-base` 与 `dsh-chat-web-search`：

```json
"dependencies": {
  "@deepseek-ai/dsh-base": "workspace:^",
  "@deepseek-ai/dsh-chat-agent": "workspace:^",
  "@deepseek-ai/dsh-chat-bots": "workspace:^",
  "@deepseek-ai/dsh-chat-web-search": "workspace:^",
  "@deepseek-ai/dsh-chat-group": "workspace:^"
},
"dsh": {
  "profile": {
    "bundles": ["@deepseek-ai/dsh-base", "@deepseek-ai/dsh-chat-agent"]
  }
}
```

改动后需重跑 `pnpm install` 以建立 workspace 链接，并验证 `dsh-base` 及其传递依赖（`dsh-session`、`dsh-llm`、`dsh-agent`、`dsh-tools`、`dsh-web` 等）能从 profile 目录解析。

## 5. 环境问题：CodeBuddy 安全删除 shim 拦截删除操作

### 现象

在 CodeBuddy IDE 集成的终端里运行时，以下操作会失败：

- `pnpm install` 的 postinstall（`install-lefthook.mjs`）报 `[safe-delete][SAFE_DELETE_BULK_CONFIRM_REQUIRED]`
- `pnpm run build` 的 `build:web`（vite）报 `[safe-delete] 操作失败: ... Error during a 'trash' operation`

### 原因

CodeBuddy IDE 通过 `NODE_OPTIONS` 注入了 `node-safe-delete-shim.cjs`（`node-language-shim.cjs`），包装了 `fs.rmSync` 等删除操作，将其改为"移到回收站"。vite 构建清空 `apps/web/dist` 目录、lefthook 安装器删除 lock 文件时，删除操作被拦截，导致命令失败。

### 结论

这是 IDE 沙箱环境注入导致的，**不是项目代码问题**。在无该 shim 的独立终端（如系统原生 PowerShell）里运行 `pnpm install`、`pnpm run build`、`pnpm tauri dev` 不会遇到此问题。

验证时可用 `pnpm install --ignore-scripts` 跳过会失败的 postinstall，确保 linking 阶段完成。

## 小结

以上改动中，第 1–4 项是跨平台的正确性修复（新增 `chat` 组时未同步更新引用、路径映射与依赖声明），并非 Windows 特有，只是本次在 Windows 上跑 desktop 开发时集中暴露。第 5 项是 IDE 环境特有的干扰。

需要注意：`.dsh-home/` 是 `.gitignore` 忽略的本地开发目录，第 4 项对 `chat-agent` profile 的修改只影响本机。若该 profile 需要在其他机器或 CI 上可用，应将其纳入版本控制，或为其建立正式的 profile 模板（目前 `PROFILE_TEMPLATES` 只有 `web` 与 `headless`）。
