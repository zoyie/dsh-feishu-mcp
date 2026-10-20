# 把飞书知识库接入 MCP：一次完整的实战记录

> 让 AI Agent 能直接读写你的飞书（Lark）知识库。
> 本文记录从零到跑通的完整过程，**包括踩过的 9 个坑**。
>
> 适用：任何支持 MCP（Model Context Protocol）的客户端。示例配置以 DSH 为例，
> 换成 Claude Desktop / Cursor / Claude Code 时只是配置文件的写法不同，CLI 部分完全通用。
>
> 验证环境：`@larksuiteoapi/lark-mcp@0.5.1`、Node 20.17.0、Windows 11、飞书（feishu.cn）

---

## 目录

- [1. 背景与目标](#1-背景与目标)
- [2. 结论速览（TL;DR）](#2-结论速览tldr)
- [3. 架构](#3-架构)
- [4. 前置条件](#4-前置条件)
- [5. 完整步骤](#5-完整步骤)
- [6. 九个坑（核心内容）](#6-九个坑核心内容)
- [7. 权限 scope 对照表](#7-权限-scope-对照表)
- [8. 工具清单与预设](#8-工具清单与预设)
- [9. 故障排查](#9-故障排查)
- [附录 A：验证 / 调用脚本](#附录-a验证--调用脚本)
- [附录 B：常见操作速查](#附录-b常见操作速查)
- [附录 C：参考链接](#附录-c参考链接)

---

## 1. 背景与目标

我的需求很具体：我在用 AI 辅助自己开发一款游戏，设计文档放在飞书知识库里。我希望：

- 我在飞书里写设计 → AI 能读到 → 据此写代码
- AI 写的技术文档 → 能直接落进飞书知识库 → 不用我手工复制粘贴

**一个分享链接是不够的。** 飞书文档需要登录态，而且前端渲染，`curl` / `fetch` 拿不到正文。所以必须走 API。

官方提供了 MCP 服务器，这条路可行——但坑不少，主要在**身份模型**上。

---

## 2. 结论速览（TL;DR）

如果你只想快速跑通，记住这五条：

| # | 结论 |
|---|---|
| 1 | 用**用户身份**（`user_access_token`），不要用应用身份 |
| 2 | 必须开通 4 个权限并**创建版本发布**：`docx:document`、`wiki:wiki`、`drive:drive`、`offline_access` |
| 3 | 用 `node.exe` + `cli.js` 绝对路径启动，**不要用 `npx`** |
| 4 | 知识库 **URL 里的数字不是 API 的 `space_id`**，必须先用 API 查 |
| 5 | 配置里必须显式写 `--token-mode user_access_token`，默认值 `auto` 会走应用身份导致失败 |

---

## 3. 架构

```
┌──────────────────┐   JSON-RPC over stdio   ┌──────────────────┐
│  MCP 客户端       │ ──────────────────────> │  lark-mcp        │
│  (AI Agent 宿主)  │ <────────────────────── │  (本地子进程)     │
└──────────────────┘      tools/list,        └────────┬─────────┘
                           tools/call                 │ HTTPS + OAuth
                                                      │ user_access_token
                                                      v
                                            ┌──────────────────────┐
                                            │  飞书开放平台 OpenAPI  │
                                            └──────────┬───────────┘
                                                       │
                                                       v
                                            ┌──────────────────────┐
                                            │  飞书知识库 / 云文档    │
                                            └──────────────────────┘
```

关键点：

- `lark-mcp` 是一个**本地 stdio 子进程**，由 MCP 客户端拉起
- 它把飞书 OpenAPI 包装成一个个 MCP 工具，工具名形如 `docx_builtin_import`
- 身份由 `--token-mode` 决定，这是整件事的核心分歧点

---

## 4. 前置条件

| 项目 | 要求 |
|---|---|
| Node.js | **≥ 20**（实测 20.17.0 可用） |
| 飞书账号 | 能创建企业自建应用的权限（通常是管理员或开发者） |
| MCP 客户端 | 支持自定义 MCP 服务器即可（DSH / Claude Desktop / Cursor / Claude Code…） |

---

## 5. 完整步骤

### 步骤 1：创建飞书自建应用

1. 打开 [飞书开放平台](https://open.feishu.cn/app) → 创建「企业自建应用」
2. 进入应用详情 → 「凭证与基础信息」，记下：
   - **App ID**（形如 `cli_xxxxxxxxxxxx`）
   - **App Secret**

> ⚠️ App Secret 只显示一次。**不要提交进 Git，不要贴到聊天/截图里。**

### 步骤 2：开通四个权限

「权限管理」里搜索并添加：

| 权限 | 用途 | 备注 |
|---|---|---|
| `docx:document` | 云文档读写 | 读正文、建文档 |
| `wiki:wiki` | 知识库读写 | 建节点、移文档 |
| `drive:drive` | 云空间上传与导入 | Markdown 导入走这里 |
| `offline_access` | 持续访问已授权的数据 | **换 refresh token 用，必须单独开通** |

> `offline_access` 最容易被忽略。它不是一个"白拿"的标准 OAuth scope，
> 在飞书里是一项独立权限。不开通就写进 `--scope`，会直接报 **错误码 20027**。

### 步骤 3：⚠️ 创建版本并发布（最容易漏的一步）

「版本管理与发布」→ 创建版本 → 发布。

**不开通版本，权限就不生效。** 前面两步全部白做。这是踩坑榜第 3 名的来源。

发布后可能需要几分钟生效。

### 步骤 4：配置 OAuth 重定向 URL

「安全设置」→ 重定向 URL，添加：

```
http://localhost:3000/callback
```

这个地址不是猜的，来自源码：`callbackUrl = http://${host}:${port}/callback`，
而 `--host` 默认 `localhost`、`--port` 默认 `3000`。

不加这条，登录时会报「重定向 URL 不匹配」。

### 步骤 5：安装 CLI（避开 npx）

```powershell
# Windows PowerShell
mkdir "$env:USERPROFILE\lark-mcp"
cd "$env:USERPROFILE\lark-mcp"
npm init -y
npm install @larksuiteoapi/lark-mcp
```

得到固定的 CLI 入口：

```
$env:USERPROFILE\lark-mcp\node_modules\@larksuiteoapi\lark-mcp\dist\cli.js
```

> macOS / Linux 同理，把路径换成 `~/lark-mcp/...` 即可。
> 为什么不用 `npx`？见[坑 #1](#坑-1windows-上-npx-直接崩-enoent)和[坑 #2](#坑-2-npx-在-windows-上-spawn-有风险)。

### 步骤 6：本地登录，拿用户令牌

```powershell
$NODE = (Get-Command node).Source
$CLI  = "$env:USERPROFILE\lark-mcp\node_modules\@larksuiteoapi\lark-mcp\dist\cli.js"

& $NODE $CLI login -a cli_你的AppID -s 你的AppSecret `
  --scope "offline_access docx:document wiki:wiki drive:drive"
```

流程：

1. 本地起 HTTP 服务 → 自动打开浏览器
2. 你用飞书账号授权
3. 令牌以 App ID 为键**存到本地**（keytar）
4. 终端显示成功

验证：

```powershell
& $NODE $CLI whoami
```

**必须检查 `scopes` 字段**：

```jsonc
"scopes": [
  "auth:user.id:read",
  "docx:document",
  "drive:drive",
  "offline_access",
  "wiki:wiki"
]
```

只有 `auth:user.id:read` 说明权限没生效 —— 回步骤 2、3 检查。

> `--scope` 用**空格**分隔（官方 README 的写法）。
> `offline_access` 决定能不能自动续期，去掉它的后果是令牌约 2 小时过期、需反复登录。

### 步骤 7：配置 MCP 客户端

以 DSH 的 `cordis.patch.yml` 为例：

```yaml
- id: mcp-feishu
  name: '@deepseek-ai/dsh-mcp-client'
  config:
    serverName: feishu
    transport: stdio
    command: 'C:\Program Files\nodejs\node.exe'          # 换成你的 node 绝对路径
    args:
      - 'C:\Users\你\lark-mcp\node_modules\@larksuiteoapi\lark-mcp\dist\cli.js'
      - 'mcp'
      - '-l'
      - 'zh'
      - '--token-mode'
      - 'user_access_token'
      - '-t'
      - 'preset.doc.default,wiki.v2.space.list,wiki.v2.spaceNode.list,wiki.v2.spaceNode.create,wiki.v2.spaceNode.moveDocsToWiki,wiki.v2.space.getNode,docx.v1.document.create,docx.v1.document.get,docx.v1.documentBlockChildren.create,docx.v1.documentBlock.patch,docx.v1.documentBlock.list'
    env:
      APP_ID: 'cli_xxxxxxxxxxxx'
      APP_SECRET: '你的AppSecret'
```

等价的其他客户端写法（Claude Desktop / Cursor 的 `mcpServers` 格式）：

```json
{
  "mcpServers": {
    "feishu": {
      "command": "node",
      "args": [
        "/path/to/lark-mcp/node_modules/@larksuiteoapi/lark-mcp/dist/cli.js",
        "mcp", "-l", "zh", "--token-mode", "user_access_token",
        "-t", "preset.doc.default"
      ],
      "env": {
        "APP_ID": "cli_xxxxxxxxxxxx",
        "APP_SECRET": "你的AppSecret"
      }
    }
  }
}
```

### 步骤 8：验证连通性

重启客户端，然后：

1. 确认工具列表出现 `mcp__feishu__*`（前缀由 `serverName` 决定）
2. 调用 `wiki_v2_space_list` —— 应该返回你能访问的知识库列表
3. **记下目标知识库的真实 `space_id`**（见[坑 #6](#坑-6知识库-url-里的数字不是-api-的-space_id)）
4. 调用 `wiki_v2_spaceNode_list` 列出节点，确认读写权限正常

工具名是**下划线形式**（`docx_builtin_import`），因为 lark-mcp 默认 `toolNameCase = snake`。

---

## 6. 九个坑（核心内容）

这一节是全文最有价值的部分。全部来自实测。

### 坑 1：Windows 上 npx 直接崩（ENOENT）

**症状**

```
npm error code ENOENT
npm error path C:\Users\<user>\AppData\Roaming\npm
npm error enoent ENOENT: no such file or directory, lstat '...\AppData\Roaming\npm'
```

**根因**

npm 的全局目录（`npm config get prefix` 的返回值）**不存在**。
npm 10.8.2 的 `npx` 遇到这种情况会直接崩溃，而不会自动创建该目录（已知 bug）。

注意 `npx --version` 是**正常**的，只有真正去执行包的时候才炸——所以很容易误判。

**修复**

```powershell
New-Item -ItemType Directory -Path "$env:APPDATA\npm" -Force
Remove-Item "$env:LOCALAPPDATA\npm-cache\_npx" -Recurse -Force -ErrorAction SilentlyContinue
```

第二条是清掉可能已损坏的 npx 缓存（曾遇到 `EPERM: rmdir` 文件锁）。

---

### 坑 2：npx 在 Windows 上 spawn 有风险

**根因**

`npx` 在 Windows 上是 `.cmd` 文件。MCP 客户端用 `child_process.spawn` 拉起子进程时，
不经过 shell 的话，`.cmd` 无法直接执行。

MCP 官方 SDK 的 `StdioClientTransport` 是否处理了这个，取决于版本。**不要赌。**

**修复**

不依赖 `npx`。把包装到固定目录，用**真实可执行文件** `node.exe` + cli.js 绝对路径：

```yaml
command: 'C:\Program Files\nodejs\node.exe'
args:
  - 'C:\Users\你\lark-mcp\node_modules\@larksuiteoapi\lark-mcp\dist\cli.js'
  - 'mcp'
```

**额外好处**：启动更快（不用每次去 registry 解析下载）、路径确定、不受 PATH 和 npm 缓存状态影响。

---

### 坑 3：「把应用加为知识库成员」这条路走不通

**症状**

知识库的「成员管理」里搜不到自己的自建应用。

**根因（两个独立证据）**

1. 飞书知识库成员管理界面**只支持「用户 / 部门 / 用户组」**，不支持应用
2. 官方 `wiki.v2.spaceMember.create` 接口的 `member_type` 字段枚举里**根本没有 `appid`**：

   ```
   openchat / userid / email / opendepartmentid / openid / unionid
   ```

   没有任何一个能表达「应用」。

飞书官方文档也说得很直接：

> 知识空间和节点都是用户的个人资源，**策略上应优先显式使用 user 身份**。

**修复**

放弃应用身份，改用用户身份（`user_access_token`）。

**附带好处**：改用用户身份后，Markdown 导入会落到**你自己的云空间**，
而不是应用那个受配额限制的云空间——原本另一个潜在故障点自动消失了。

---

### 坑 4：`offline_access` 要单独开通（错误码 20027）

**症状**

登录时浏览器页面：

```
当前应用权限不足，请联系应用管理员，前往开发者后台 开通以下权限
  · 持续访问已授权的数据：offline_access
错误码：20027
```

**根因**

`offline_access` 在飞书里**不是标准 OAuth scope，而是一项需要单独开通的权限**。

**修复**

去「权限管理」搜「持续访问已授权的数据」并添加，然后**创建版本发布**。

**排查技巧**：20027 会点名缺哪个权限，但它是**逐个暴露**的——一次只报第一个。
所以不要一个一个试，**一次性把四个权限全开通**。

---

### 坑 5：`--token-mode` 默认值会走应用身份

**症状**

权限都开了，登录也成功，但工具调用返回权限错误。

**根因（源码逻辑）**

```js
function getShouldUseUAT(tokenMode = AUTO, useUAT) {
  switch (tokenMode) {
    case USER_ACCESS_TOKEN:  return true;   // 总是用用户身份
    case TENANT_ACCESS_TOKEN: return false; // 总是用应用身份
    case AUTO:               return useUAT; // 由每次调用的参数决定，默认 false → 应用身份
  }
}
```

`auto` 是默认值，而 `useUAT` 参数默认 `false`，于是走了应用身份。

**修复**

配置里显式写：

```
--token-mode user_access_token
```

---

### 坑 6：知识库 URL 里的数字不是 API 的 `space_id`

**症状**

从浏览器地址栏拿到的数字：

```
https://<租户>.feishu.cn/wiki/space/6891234567890123456
                                   ^^^^^^^^^^^^^^^^^^^
                                   这个不是 space_id
```

但 `wiki.v2.space.list` 返回的完全不同：

```json
{
  "items": [
    { "name": "我的项目知识库", "space_id": "7012345678901234567" }
  ]
}
```

**后果**

如果用 URL 里那个数字去做写操作（建节点、移文档），**会全部失败**。

**修复**

永远先查再写：

```
wiki_v2.space.list  →  按 name 精确匹配  →  取真实 space_id
```

不要从 URL 猜，也不要凭记忆。

---

### 坑 7：`moveDocsToWiki` 返回 `task_id` 不是失败

**症状**

```json
{"task_id": "7695426225312894218-f3e1f74f80e76d1672aa4810500ee5c6e586eb97"}
```

同一批 10 个文档，9 个返回 `{"wiki_token":"..."}`，1 个返回 `task_id`。脚本把它判成失败了。

**真相**

**节点其实已经建好了。** 这个接口可能是同步也可能是异步的。

**修复**

不要只看返回值。**用 `wiki.v2.spaceNode.list` 判断真实结果。**

```powershell
# 批处理里应该在失败后复查，而不是直接判定失败
wiki_v2.spaceNode.list → 看目标标题在不在
```

---

### 坑 8：Markdown 导入后标题重复

**症状**

文档标题是 `08 - 开发日志`，正文第一行也是一个大标题 `08 - 开发日志`。

**根因**

`docx.builtin.import` 用 `file_name` 作为文档标题，同时把 Markdown 里的 `# 一级标题` 
原样保留在正文。两者相同时就重复了。

**修复**

导入前剥掉开头的一级标题：

```js
function stripH1(md) {
  return md.replace(/^\s*#\s+[^\n]*\n+/, '');
}
```

---

### 坑 9：没有删除节点 / 重排节点的接口

**事实**

这个 MCP 工具集的 wiki 能力只有：

```
space.create / get / getNode / list
spaceMember.create / delete / list
spaceNode.copy / create / list / move / moveDocsToWiki / updateTitle
spaceSetting.update
task.get
```

**没有** `spaceNode.delete`，`spaceNode.move` 也只改父节点、不改排序。

**后果**

- 删节点：只能在飞书界面手动操作
- **节点顺序 = 节点创建时间顺序**，事后无法用 API 重排

**教训**

**批量发布必须按目标顺序依次创建。** 我当时先测试了一个节点（08），
结果它永久占据了第 2 位，只能手动拖。

---

## 7. 权限 scope 对照表

| Scope | 用途 | 不开通的后果 |
|---|---|---|
| `docx:document` | 云文档读写（读正文、建文档） | 读/写文档失败 |
| `wiki:wiki` | 知识库读写（建节点、移文档） | 知识库操作失败 |
| `drive:drive` | 云空间上传、导入任务 | Markdown 导入失败 |
| `offline_access` | 换 refresh token | 登录报 20027；令牌 2 小时过期需重登 |
| `auth:user.id:read` | 基础身份 | OAuth 登录自带，通常不用管 |

> 如果你还需要给别人授权协作者，加 `drive.v1.permissionMember.create` 对应的权限。

---

## 8. 工具清单与预设

### 内置预设

`-t` 支持预设名与具体工具名**混写**（逗号或空格分隔）：

| 预设 | 内容方向 |
|---|---|
| `preset.light` | 精简集（IM + Bitable + Doc + Contact 各取几个） |
| `preset.default` | IM + Bitable + Doc + Contact 常用集 |
| `preset.im.default` | 消息与群组 |
| `preset.base.default` | 多维表格 |
| `preset.base.batch` | 多维表格（含批量写） |
| `preset.doc.default` | **云文档 + 知识库读取** |
| `preset.task.default` | 任务 |
| `preset.calendar.default` | 日历 |

### `preset.doc.default` 具体包含

```
docx.v1.document.rawContent      读文档正文
docx.builtin.import              导入 Markdown 生成文档   ← 写入关键
docx.builtin.search              搜索文档
drive.v1.permissionMember.create 添加协作者
wiki.v2.space.getNode            解析知识库节点
wiki.v1.node.search              搜索知识库节点
```

⚠️ **它不含建节点和搬运文档的工具。** 想写入知识库必须额外补：

```
wiki.v2.space.list
wiki.v2.spaceNode.list
wiki.v2.spaceNode.create
wiki.v2.spaceNode.moveDocsToWiki      ← 写入关键
docx.v1.document.create
docx.v1.document.get
docx.v1.documentBlockChildren.create
docx.v1.documentBlock.patch
docx.v1.documentBlock.list
```

### 写入知识库的两步流程

```
第 1 步  docx.builtin.import
         入参 { data: { markdown, file_name } }
         出参 { result: { token, url, job_status } }     ← token 就是 obj_token

第 2 步  wiki.v2.spaceNode.moveDocsToWiki
         入参 { path: { space_id }, data: { obj_type: "docx", obj_token } }
         出参 { wiki_token }  或  { task_id }（异步，不是失败）
```

读回来：

```
wiki.v2.space.getNode   { params: { token: <node_token> } }  → node.obj_token
docx.v1.document.rawContent  { path: { document_id: <obj_token> } }
```

---

## 9. 故障排查

### 工具列表里没有 `mcp__feishu__*`

1. 检查配置的 YAML/JSON 缩进（YAML 对缩进敏感）
2. 确认两个路径都真实存在：
   ```powershell
   Test-Path 'C:\Program Files\nodejs\node.exe'
   Test-Path "$env:USERPROFILE\lark-mcp\node_modules\@larksuiteoapi\lark-mcp\dist\cli.js"
   ```
3. 手动跑一次看报错：
   ```powershell
   & $NODE $CLI mcp -a cli_xxx -s xxx --token-mode user_access_token --debug
   ```

### 错误码 20027：应用权限不足

按报错点名的权限去「权限管理」添加，**然后创建版本发布**，再重新登录。

### 错误码 99991679 / 99991668

- `99991679`：`user_access_token` 缺少所需 scope → 补权限 + 发布版本 + **重新登录**
- `99991668`：令牌无效或过期 → 重新登录

### 令牌反复过期

检查 `scopes` 里有没有 `offline_access`。有它才能自动续期。

> 一个实现细节：续期时代码取 secret 的写法是 `extra?.app_secret || options.appSecret`，
> 而登录时存的字段名是 `appSecret`（驼峰）——所以实际上**永远回退到配置文件里的 secret**。
> 含义：**改了 App Secret 后必须同步更新配置**，否则 2 小时后续期必然失败。

### 提示无权限但权限都开了

- 权限开通后**没有创建版本发布**
- 企业开了权限审批，还没审批通过
- 只勾了「应用身份」没勾「用户身份」

---

## 附录 A：验证 / 调用脚本

一个最小可用的 MCP stdio 调用器，用于脚本化操作（也适合写进 CI 或批处理）。

依赖：`yaml` 包（`npm i yaml`），用于从 MCP 配置里读凭据。

```js
#!/usr/bin/env node
/**
 * lark-call.mjs —— 调用飞书 MCP 服务器上的一个工具
 *
 * 用法:
 *   node lark-call.mjs <toolName> [jsonArgs | @argsFile]
 *
 * 例:
 *   node lark-call.mjs wiki_v2_space_list '{"params":{"page_size":50}}'
 *   node lark-call.mjs docx_builtin_import @args.json
 */
import fs from 'node:fs';
import { spawn } from 'node:child_process';
import { createRequire } from 'node:module';

// 改成你客户端配置文件的位置
const CONFIG = 'C:/Users/你/.dsh/profiles/desktop/cordis.patch.yml';
const ENTRY_ID = 'mcp-feishu';

const require = createRequire(CONFIG);
const YAML = require('yaml');

const toolName = process.argv[2];
const argSpec  = process.argv[3];
const toolArgs = !argSpec
  ? {}
  : argSpec.startsWith('@')
    ? JSON.parse(fs.readFileSync(argSpec.slice(1), 'utf8'))
    : JSON.parse(argSpec);

if (!toolName) {
  console.error('用法: node lark-call.mjs <toolName> [jsonArgs | @argsFile]');
  process.exit(2);
}

const doc = YAML.parse(fs.readFileSync(CONFIG, 'utf8'));
const entry = doc.find((x) => x && x.id === ENTRY_ID);
if (!entry) { console.error('配置里找不到 ' + ENTRY_ID); process.exit(2); }

const { command, args, env: envCfg } = entry.config;
const proc = spawn(command, args, {
  env: { ...process.env, APP_ID: envCfg.APP_ID, APP_SECRET: envCfg.APP_SECRET },
  stdio: ['pipe', 'pipe', 'pipe'],
});

let buf = '';
const responses = [];
proc.stdout.on('data', (d) => {
  buf += d.toString();
  let i;
  while ((i = buf.indexOf('\n')) >= 0) {
    const line = buf.slice(0, i).trim();
    buf = buf.slice(i + 1);
    if (!line) continue;
    try { responses.push(JSON.parse(line)); } catch { /* 忽略非 JSON 行 */ }
  }
});

const send  = (o) => proc.stdin.write(JSON.stringify(o) + '\n');
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

function finish(code, payload) {
  try { proc.kill(); } catch { /* 已退出 */ }
  if (payload !== undefined) console.log(payload);
  process.exit(code);
}

(async () => {
  await sleep(700);
  send({ jsonrpc: '2.0', id: 1, method: 'initialize', params: {
    protocolVersion: '2024-11-05', capabilities: {},
    clientInfo: { name: 'lark-call', version: '1' },
  }});

  for (let i = 0; i < 60 && !responses.find((r) => r.id === 1); i++) await sleep(100);
  if (!responses.find((r) => r.id === 1)) finish(1, '初始化失败');

  send({ jsonrpc: '2.0', method: 'notifications/initialized' });
  await sleep(150);
  send({ jsonrpc: '2.0', id: 2, method: 'tools/call',
         params: { name: toolName, arguments: toolArgs } });

  for (let i = 0; i < 600 && !responses.find((r) => r.id === 2); i++) await sleep(100);
  const res = responses.find((r) => r.id === 2);
  if (!res) finish(1, '调用超时（60 秒）');
  if (res.error) finish(1, '协议错误: ' + JSON.stringify(res.error));

  const text = (res.result?.content || []).map((c) => c.text).join('\n');
  if (res.result?.isError) finish(1, '工具返回错误:\n' + text);
  finish(0, text);
})();
```

### 批量发布脚本的核心逻辑

```js
// 1) 剥掉一级标题，避免与文档标题重复
const md = fs.readFileSync(file, 'utf8').replace(/^\s*#\s+[^\n]*\n+/, '');

// 2) 导入
const imp = JSON.parse(await call('docx_builtin_import', {
  data: { markdown: md, file_name: title },
}));
const objToken = imp.result.token;

// 3) 搬进知识库
const mv = JSON.parse(await call('wiki_v2_spaceNode_moveDocsToWiki', {
  path: { space_id: SPACE_ID },
  data: { obj_type: 'docx', obj_token: objToken },
}));

// 4) 注意：mv 可能是 { task_id }，不是失败 —— 用 spaceNode.list 复查
```

---

## 附录 B：常见操作速查

```powershell
$NODE = (Get-Command node).Source
$CLI  = "$env:USERPROFILE\lark-mcp\node_modules\@larksuiteoapi\lark-mcp\dist\cli.js"

# 查看登录状态
& $NODE $CLI whoami

# 重新登录（指定 scope）
& $NODE $CLI login -a cli_xxx -s xxx --scope "offline_access docx:document wiki:wiki drive:drive"

# 退出登录
& $NODE $CLI logout

# 列出能访问的知识库
& $NODE lark-call.mjs wiki_v2_space_list '{"params":{"page_size":50}}'

# 列出知识库的一级节点
& $NODE lark-call.mjs wiki_v2_spaceNode_list '{"path":{"space_id":"<space_id>"},"params":{"page_size":50}}'

# 节点 token → obj_token
& $NODE lark-call.mjs wiki_v2_space_getNode '{"params":{"token":"<node_token>"}}'

# 读文档正文
& $NODE lark-call.mjs docx_v1_document_rawContent '{"path":{"document_id":"<obj_token>"}}'

# 搜索云文档
& $NODE lark-call.mjs docx_builtin_search '{"data":{"search_key":"关键词"}}'
```

---

## 附录 C：参考链接

- [飞书开放平台 · 开发者后台](https://open.feishu.cn/app)
- [飞书 MCP 集成文档（本地调用 OpenAPI MCP）](https://open.feishu.cn/document/uAjLw4CM/ukTMukTMukTM/mcp_integration/mcp_installation)
- [飞书 MCP 高级配置](https://open.feishu.cn/document/uAjLw4CM/ukTMukTMukTM/mcp_integration/advanced-configuration?lang=zh-CN)
- [lark-openapi-mcp（官方仓库）](https://github.com/larksuite/lark-openapi-mcp)
- [@larksuiteoapi/lark-mcp（npm）](https://www.npmjs.com/package/@larksuiteoapi/lark-mcp)
- [如何为应用开通云文档相关资源的权限](https://open.feishu.cn/document/faq/trouble-shooting/how-to-add-permissions-to-app)
- [Model Context Protocol](https://modelcontextprotocol.io/)

---

## 写在最后

整件事的难点不在 API，而在**身份模型**。

飞书有两套身份：应用身份（`tenant_access_token`）和用户身份（`user_access_token`）。
直觉上"给应用授权"更规范，但在这个场景下它是错的——因为知识库是**个人资源**，
而飞书的成员体系压根不把「应用」当成一种成员类型。

一旦接受"我是代用户操作他个人的资料"，后面所有问题都顺了：
用用户身份、开 `offline_access`、走 OAuth 登录、当自己是那个人的键盘。

另外一个教训：**不要相信 URL 里的 ID**。所有 ID 一律通过 API 查询，按名称匹配。
这一条在几乎所有开放平台上都适用。

---

发布到 GitHub 前请检查：

- [ ] 替换所有 `cli_xxxxxxxxxxxx` 为你自己的占位符或删掉
- [ ] 确认没有任何 App Secret 残留
- [ ] 替换个人路径（`C:\Users\你\...`）
- [ ] 如果不想公开你的知识库名称，把示例改成通用名称
- [ ] 考虑加一个 `LICENSE`
