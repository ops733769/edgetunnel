# edgetunnel 2.0 — Code Wiki

> 本文档为 `cmliu/edgetunnel` 仓库的结构化代码 Wiki，涵盖项目整体架构、主要模块职责、关键类与函数说明、依赖关系以及运行方式等关键信息，便于开发者快速理解、二次开发与运维部署。

---

## 目录

1. [项目概述](#1-项目概述)
2. [仓库结构](#2-仓库结构)
3. [整体架构](#3-整体架构)
4. [运行时请求流转](#4-运行时请求流转)
5. [主要模块职责](#5-主要模块职责)
6. [关键函数说明](#6-关键函数说明)
7. [配置数据结构（config_JSON）](#7-配置数据结构config_json)
8. [KV 存储模型](#8-kv-存储模型)
9. [环境变量与依赖关系](#9-环境变量与依赖关系)
10. [订阅生成与协议适配](#10-订阅生成与协议适配)
11. [代理转发机制](#11-代理转发机制)
12. [项目运行方式](#12-项目运行方式)
13. [关键约束与注意事项](#13-关键约束与注意事项)

---

## 1. 项目概述

**edgetunnel** 是一个基于 Cloudflare Workers / Pages 平台的边缘计算隧道方案。它通过 WebSocket 协议承载 VLESS / Trojan 流量，在 CF 边缘节点完成流量解密与转发，同时内置可视化后台管理面板、订阅自动生成、多客户端协议适配（Clash / Sing-box / Surge / QuantumultX / Loon）、SOCKS5/HTTP 链式代理、ProxyIP 反代兜底、TG 日志推送、CF 用量查询等能力。

- **运行平台**：Cloudflare Workers / Pages
- **核心语言**：JavaScript（单文件 ES Module）
- **入口文件**：[_worker.js](file:///workspace/_worker.js)
- **License**：GPL-2.0（见 [LICENSE](file:///workspace/LICENSE)）
- **配置入口**：[wrangler.toml](file:///workspace/wrangler.toml)

---

## 2. 仓库结构

```
edgetunnel/
├── _worker.js                    # 主程序（唯一核心代码文件，ES Module 入口）
├── wrangler.toml                 # Wrangler / Pages 部署配置
├── img.png                       # README 后台截图
├── LICENSE                       # GPL-2.0 协议
├── README.md                     # 项目说明与部署教程
└── .github/
    └── workflows/
        └── sync.yml              # 每日自动从上游 cmliu/edgetunnel 同步 Fork
```

仓库采用 **单文件架构**：所有业务逻辑集中于 [_worker.js](file:///workspace/_worker.js)（约 2116 行）。前端管理页面（`/admin`、`/login` 等）通过 `fetch` 远程拉取静态页面 `https://edt-pages.github.io` 提供，本仓库不包含前端静态资源。

### 关键文件说明

#### [wrangler.toml](file:///workspace/wrangler.toml)

```toml
name = "v20251104"
main = "_worker.js"
compatibility_date = "2025-11-04"
keep_vars = true

#[[kv_namespaces]]
#binding = "KV"            #KV绑定名默认不可修改
#id = ""                   #KV数据库id
```

- `main` 指定 `_worker.js` 为入口
- `keep_vars = true` 保留环境变量
- KV 绑定默认变量名为 `KV`（不可改名）

#### [.github/workflows/sync.yml](file:///workspace/.github/workflows/sync.yml)

通过 `aormsby/Fork-Sync-With-Upstream-action@v3.4` 每日（`cron: "0 0 * * *"`）从上游 `cmliu/edgetunnel` 的 `main` 分支同步至 Fork 仓库，仅当当前仓库为 Fork 时执行。

---

## 3. 整体架构

### 3.1 分层架构

```
┌──────────────────────────────────────────────────────────────┐
│                    Cloudflare Edge (Worker)                   │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │            入口层: export default { fetch }             │ │
│  │   URL 解析 / UA 提取 / 鉴权 / 路由分发 / 兜底伪装页      │ │
│  └─────────────────────────────────────────────────────────┘ │
│                            │                                  │
│       ┌────────────────────┼────────────────────┐            │
│       ▼                    ▼                    ▼            │
│  ┌─────────┐         ┌──────────┐         ┌──────────┐       │
│  │  HTTP   │         │   WS     │         │  反代伪装 │       │
│  │ 路由层  │         │ 代理层   │         │  页面层   │       │
│  └─────────┘         └──────────┘         └──────────┘       │
│       │                    │                    │            │
│       ▼                    ▼                    ▼            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  业务功能层                                            │   │
│  │  - 后台管理 (login/admin/POST 接口)                   │   │
│  │  - 订阅生成 (sub/mixed/clash/singbox/surge)          │   │
│  │  - 协议解析 (VLESS / Trojan)                          │   │
│  │  - 代理转发 (ProxyIP / SOCKS5 / HTTP / Direct)       │   │
│  │  - 配置管理 (KV 读写 / config_JSON)                   │   │
│  │  - 日志 / TG 推送 / CF 用量查询                       │   │
│  └──────────────────────────────────────────────────────┘   │
│                            │                                  │
│                            ▼                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  存储层: Cloudflare KV (config.json / cf.json /       │   │
│  │          tg.json / ADD.txt / log.json)                │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
                            │
                            ▼
                外部依赖: DoH(1.1.1.1) / CF API /
                Telegram API / 优选IP API / 订阅转换后端 /
                静态页面(edt-pages.github.io)
```

### 3.2 模块划分（按代码区域）

[_worker.js](file:///workspace/_worker.js) 内部按注释块划分为以下功能区域：

| 行号区间 | 区域 | 职责 |
|---------|------|------|
| 1–8 | 全局状态 | 模块级变量、SOCKS5 白名单、静态页面 URL |
| 9–369 | 主程序入口 | `export default { fetch }`，HTTP 路由与请求分发 |
| 370–718 | WS 传输数据 | VLESS/Trojan 协议解析、TCP/UDP 转发、流连接 |
| 719–788 | SOCKS5/HTTP 代理 | `socks5Connect` / `httpConnect` 实现 |
| 789–1210 | 订阅热补丁 | Clash / Singbox / Surge 配置文件后处理 |
| 1211–1296 | 日志与工具 | 日志记录、TG 推送、MD5MD5、敏感信息掩码 |
| 1298–1358 | 路径与 ECH | 随机路径、域名批量替换、ECH 配置查询 |
| 1359–1518 | 配置加载 | `读取config_JSON` 默认配置与合并逻辑 |
| 1519–1696 | 优选 IP | 随机 IP 生成、优选 API 请求、地址解析 |
| 1697–1790 | 反代参数解析 | URL 路径/查询参数提取 ProxyIP/SOCKS5 |
| 1792–1849 | CF 用量查询 | `getCloudflareUsage` 调用 CF GraphQL API |
| 1850–1969 | 哈希与反代解析 | sha224 实现、ProxyIP 域名 DoH 解析 |
| 1971–1994 | SOCKS5 验证 | 代理可用性检测 |
| 1996–2116 | HTML 伪装页 | `nginx()` / `html1101()` 内置页面 |

---

## 4. 运行时请求流转

### 4.1 总入口（`fetch` handler）

入口位于 [_worker.js:9-369](file:///workspace/_worker.js)。请求处理流程：

```
请求到达 fetch(request, env, ctx)
        │
        ▼
解析 URL / UA / 提取管理员密码(env.ADMIN 等)
计算 userID（env.UUID 或基于 MD5MD5 派生 UUIDv4）
        │
        ▼
是否 WebSocket 升级请求? (Upgrade: websocket)
        │
        ├── 是 ──► 反代参数获取(request) ──► 处理WS请求(request, userID)
        │
        └── 否 ──► HTTP 路由分发
                    │
                    ├─ http: 协议 ──► 301 重定向到 https
                    │
                    ├─ 无 ADMIN 密码 ──► 拉取 /noADMIN 静态页
                    │
                    ├─ 已绑定 KV ──► 按访问路径路由：
                    │     ├─ {KEY}            快速订阅 → 302 到 /sub?token=...
                    │     ├─ /login           登录页/登录校验
                    │     ├─ /admin/*         后台管理（需 cookie 鉴权）
                    │     │     ├─ /admin/log.json         读日志
                    │     │     ├─ /admin/getCloudflareUsage  CF 用量
                    │     │     ├─ /admin/getADDAPI         验证优选 API
                    │     │     ├─ /admin/check             SOCKS5 检测
                    │     │     ├─ /admin/init              重置配置
                    │     │     ├─ POST /admin/config.json  保存主配置
                    │     │     ├─ POST /admin/cf.json      保存 CF 配置
                    │     │     ├─ POST /admin/tg.json      保存 TG 配置
                    │     │     ├─ POST /admin/ADD.txt      保存优选 IP
                    │     │     ├─ GET  /admin/config.json  读主配置
                    │     │     ├─ GET  /admin/ADD.txt      读优选 IP
                    │     │     └─ GET  /admin/cf.json      读 request.cf
                    │     ├─ /sub              订阅生成
                    │     ├─ /locations        反代 CF 测速点列表
                    │     ├─ /robots.txt       返回 Disallow
                    │     ├─ /logout 或 UUID   清 cookie 跳登录
                    │     └─ 其他              返回伪装页
                    │
                    └─ 未绑定 KV 且无 envUUID ──► 拉取 /noKV 静态页
        │
        ▼
非以上分支（无 KV / 非管理路径）：
  根据 env.URL 决定伪装页：
    ├─ 'nginx'      ► nginx() 内置欢迎页
    ├─ '1101'       ► html1101() CF 错误页伪装
    └─ 其他 URL     ► fetch 反代该站点内容
```

### 4.2 鉴权机制

- **管理员密码**：`env.ADMIN` 优先，回退链为 `ADMIN → admin → PASSWORD → password → pswd → TOKEN → KEY → UUID`（见 [_worker.js:14](file:///workspace/_worker.js)）。
- **登录 Cookie**：登录成功后写入 `auth=<MD5MD5(UA+加密秘钥+管理员密码)>; Max-Age=86400; HttpOnly`。
- **订阅 TOKEN**：`MD5MD5(host + userID)`，作为 `/sub?token=` 校验值。
- **加密秘钥**：`env.KEY`，默认值 `'勿动此默认密钥，有需求请自行通过添加变量KEY进行修改'`；当 KEY 与默认值不同时，访问 `/{KEY}` 即触发快速订阅重定向。

### 4.3 WebSocket 代理流程

```
WS 升级请求 ──► 反代参数获取(request)
                  │  解析 URL 中的 proxyip= / socks5= / http= / globalproxy
                  ▼
            处理WS请求(request, userID)
                  │
                  ▼
        构造 WebSocketPair, 接受 serverSock
        将 sec-websocket-protocol 头作为 earlyData
        readable.pipeTo(WritableStream)
                  │
                  ▼
        首包判断协议（Trojan 头 0x0d 0x0a 在偏移 56/57）
                  │
            ┌─────┴─────┐
            ▼           ▼
      解析木马请求   解析魏烈思请求(VLESS)
            │           │
            └─────┬─────┘
                  ▼
          forwardataTCP(host, port, rawData, ws, ...)
                  │
        ┌─────────┼──────────┐
        ▼         ▼          ▼
   SOCKS5/HTTP   ProxyIP    直连
   (白名单/全局)  反代兜底    connect()
```

---

## 5. 主要模块职责

### 5.1 入口与路由模块

负责解析请求、鉴权、按 URL 路径分发到对应处理器。包含 HTTP 路由（登录/后台/订阅/伪装页）与 WebSocket 升级判断两条主线。

### 5.2 后台管理模块

通过远程静态页面 `https://edt-pages.github.io/admin`、`/login` 提供前端 UI，Worker 侧提供 RESTful 接口完成配置 CRUD、日志读取、CF 用量查询、优选 API 验证、SOCKS5 检测。

### 5.3 订阅生成模块

依据请求 UA / 查询参数自动识别客户端类型（mixed / clash / singbox / surge / quanx / loon），生成对应的订阅内容。支持两种生成路径：

- **本地生成**（`config_JSON.优选订阅生成.local = true`）：基于 KV 中的 `ADD.txt` 或随机生成的 CF 优选 IP。
- **优选订阅生成器**（外部 `SUB` host）：拉取生成器返回的订阅并解析节点。

非 mixed 类型通过订阅转换后端（`SUBAPI`）转换，并对其结果执行协议热补丁。

### 5.4 协议解析与转发模块

- **VLESS 解析**：`解析魏烈思请求` 解析版本、UUID 校验、命令（TCP/UDP）、地址类型（IPv4/Domain/IPv6）、端口。
- **Trojan 解析**：`解析木马请求` 校验 sha224 密码、解析 SOCKS5 风格地址。
- **TCP 转发**：`forwardataTCP` 优先直连，失败回退 ProxyIP 反代或 SOCKS5/HTTP 代理。
- **UDP 转发**：仅支持 53 端口 DNS 查询，转发到 `8.8.4.4:53`。

### 5.5 代理链模块

- **ProxyIP**：通过 `解析地址端口` 将 ProxyIP（域名/IP/`.william` TXT/`.tp端口`）解析为地址端口数组，并按目标域名+UUID 生成随机种子洗牌，取前 8 个尝试连接。
- **SOCKS5**：`socks5Connect` 实现 RFC 1928 握手与用户名密码认证。
- **HTTP CONNECT**：`httpConnect` 通过 HTTP/1.1 CONNECT 建立隧道。
- **白名单**：`SOCKS5白名单` 控制哪些目标域名强制走 SOCKS5，可通过 `env.GO2SOCKS5` 覆盖。

### 5.6 配置与存储模块

`读取config_JSON` 负责从 KV 读取 `config.json`，若不存在或重置则写入默认配置，并合并 `tg.json`、`cf.json`，最终返回完整 `config_JSON`。所有配置通过 KV 持久化。

### 5.7 日志与通知模块

`请求日志记录` 将请求信息（类型/IP/ASN/CC/URL/UA/时间）追加到 KV 的 `log.json`，单文件容量上限 4MB（FIFO 滚动）。当 TG 启用时，`sendMessage` 通过 Telegram Bot API 推送日志通知。

### 5.8 伪装页模块

当请求未命中任何业务路由时，根据 `env.URL` 返回：

- `nginx`：内置 nginx 欢迎页（`nginx()`）。
- `1101`：模拟 CF Worker 1101 错误页（`html1101()`）。
- 其他 URL：反代目标站点，并将响应中的目标 host 替换为当前 host。

---

## 6. 关键函数说明

### 6.1 入口与路由

#### `export default { async fetch(request, env, ctx) }` — [_worker.js:9](file:///workspace/_worker.js)

Worker 唯一入口。完成 URL 解析、密码与 UUID 派生、KV 绑定检测、HTTP/WS 路由分发、伪装页兜底。返回 `Response` 对象。

### 6.2 WebSocket 与协议解析

#### `async function 处理WS请求(request, yourUUID)` — [_worker.js:371](file:///workspace/_worker.js)

构造 `WebSocketPair`，接受 server 端，将首包分流到 Trojan 或 VLESS 解析器，后续数据写入远端 socket。返回 `Response(null, { status: 101, webSocket: clientSock })` 完成升级。

#### `function 解析木马请求(buffer, passwordPlainText)` — [_worker.js:426](file:///workspace/_worker.js)

解析 Trojan 协议首包：校验 sha224 密码（56 字节 + CRLF），解析 SOCKS5 风格地址（atype 1/3/4 对应 IPv4/Domain/IPv6）与端口。返回 `{ hasError, addressType, port, hostname, rawClientData }`。

#### `function 解析魏烈思请求(chunk, token)` — [_worker.js:485](file:///workspace/_worker.js)

解析 VLESS 协议首包：版本(1B)、UUID(16B，与 token 比对)、附加选项长度(1B)、命令(1B：1=TCP, 2=UDP)、端口(2B)、地址类型与地址。返回 `{ hasError, addressType, port, hostname, isUDP, rawIndex, version }`。

#### `async function forwardataTCP(host, portNum, rawData, ws, respHeader, remoteConnWrapper, yourUUID)` — [_worker.js:520](file:///workspace/_worker.js)

TCP 转发核心。内含两个闭包：

- `connectDirect(address, port, data, 所有反代数组, 反代兜底)`：优先尝试反代数组中的地址（轮询+缓存索引+1s 连接超时），全部失败时根据 `反代兜底` 决定直连或抛错。
- `connecttoPry()`：根据 `启用SOCKS5反代` 类型走 SOCKS5 / HTTP / ProxyIP 反代。

外层根据是否启用 SOCKS5 全局或白名单匹配决定直连优先还是代理优先。

#### `async function forwardataudp(udpChunk, webSocket, respHeader)` — [_worker.js:601](file:///workspace/_worker.js)

将 UDP DNS 查询转发至 `8.8.4.4:53`，响应回写 WebSocket（首包附加 VLESS 响应头）。

#### `async function connectStreams(remoteSocket, webSocket, headerData, retryFunc)` — [_worker.js:640](file:///workspace/_worker.js)

将远端 socket 的 readable pipe 到 WebSocket。无数据且提供 `retryFunc` 时触发重试。

#### `function makeReadableStr(socket, earlyDataHeader)` — [_worker.js:667](file:///workspace/_worker.js)

构造 ReadableStream，处理 `sec-websocket-protocol` 中的 base64 early data（v2ray ws 0-RTT）。

#### `function isSpeedTestSite(hostname)` — [_worker.js:692](file:///workspace/_worker.js)

屏蔽 `speed.cloudflare.com`（base64 编码）及其子域名，防止测速流量。

### 6.3 SOCKS5 / HTTP 代理

#### `async function socks5Connect(targetHost, targetPort, initialData)` — [_worker.js:720](file:///workspace/_worker.js)

实现 SOCKS5 客户端：方法协商（0x00/0x02）→ 用户名密码认证（如需）→ CONNECT 请求（atype 0x03 域名）→ 发送 initialData。返回已建立的 `socket`。

#### `async function httpConnect(targetHost, targetPort, initialData)` — [_worker.js:756](file:///workspace/_worker.js)

通过 HTTP/1.1 `CONNECT` 建立隧道，解析响应状态码（2xx 成功），支持 `Proxy-Authorization: Basic` 认证。

#### `async function SOCKS5可用性验证(代理协议, 代理参数)` — [_worker.js:1971](file:///workspace/_worker.js)

通过代理请求 `check.socks5.090227.xyz/cdn-cgi/trace`，返回 `{ success, proxy, ip, loc, responseTime }` 用于后台检测代理可用性。

### 6.4 订阅热补丁

#### `function Clash订阅配置文件热补丁(...)` — [_worker.js:789](file:///workspace/_worker.js)

对 Clash YAML 订阅进行后处理：规范化 `mode: rule`、注入 DNS 配置块、根据 ECH 启用情况在 `dns:` 块内插入 `nameserver-policy` 指向 ECH DNS 与 `doh.cm.edu.kg`。处理 nameserver-policy 已存在与不存在两种情形。

#### `function Singbox订阅配置文件热补丁(...)` — [_worker.js:1009](file:///workspace/_worker.js)

对 Sing-box JSON 订阅进行兼容性迁移：DNS `1.1.1.1→8.8.8.8`、TUN 入站 `inet4_address/inet6_address → address`、`inet4_route_address → route_address` 等 1.10.0+ 字段迁移，并按需注入 ECH 配置。

#### `function Surge订阅配置文件热补丁(content, url, config_JSON)` — [_worker.js:1192](file:///workspace/_worker.js)

对 Surge 订阅补全 Trojan 节点的 `ws=true, ws-path, ws-headers`，并在头部插入 `#!MANAGED-CONFIG` 自动更新声明。

### 6.5 配置管理

#### `async function 读取config_JSON(env, hostname, userID, 重置配置 = false)` — [_worker.js:1359](file:///workspace/_worker.js)

配置加载与合并的核心函数。流程：

1. 构造 `默认配置JSON`（见 [§7](#7-配置数据结构config_json)）。
2. 从 KV 读取 `config.json`，不存在或重置时写入默认值。
3. 合并 `env.HOST`、`env.PATH`、`env.HOSTS` 等环境变量覆盖。
4. 计算完整节点路径（拼接 PATH + 反代参数 + 0RTT `ed=2560`）。
5. 生成节点 `LINK`。
6. 合并 `tg.json`（敏感信息掩码）与 `cf.json`（调用 `getCloudflareUsage` 查询用量）。
7. 返回完整 `config_JSON`，附带 `加载时间`。

### 6.6 日志与通知

#### `async function 请求日志记录(env, request, 访问IP, 请求类型, config_JSON)` — [_worker.js:1211](file:///workspace/_worker.js)

将请求追加到 KV `log.json`，容量上限 4MB（超出时 FIFO 出队）。对非 `Get_SUB` 类型做 30 分钟内同 IP/URL/UA 去重。当 TG 启用时同步推送。

#### `async function sendMessage(BotToken, ChatID, 日志内容, config_JSON)` — [_worker.js:1244](file:///workspace/_worker.js)

构造 HTML 格式消息（类型/IP/位置/ASN/域名/路径/UA/时间/用量），调用 `https://api.telegram.org/bot{Token}/sendMessage` 推送。

#### `function 掩码敏感信息(文本, 前缀长度 = 3, 后缀长度 = 2)` — [_worker.js:1273](file:///workspace/_worker.js)

将 API Key 等敏感信息中间部分替换为 `*`，仅保留前后若干字符，用于后台展示。

### 6.7 优选 IP 与反代解析

#### `async function 生成随机IP(request, count = 16, 指定端口 = -1)` — [_worker.js:1519](file:///workspace/_worker.js)

根据请求 ASN（9808 移动 / 4837,17623,17816 联通 / 4134 电信）选择对应 CF CIDR 列表，从中随机生成 `count` 个 IP，端口从 `[443,2053,2083,2087,2096,8443]` 随机或指定。返回 `[IP数组, 文本]`。

#### `async function 请求优选API(urls, 默认端口 = '443', 超时时间 = 3000)` — [_worker.js:1577](file:///workspace/_worker.js)

并发请求多个优选 IP API（3s 超时），自动识别响应格式（Base64 明文 LINK / CSV / 纯 IP 列表），返回 `[IP集合, 明文LINK, 待转换订阅URL]`。支持 GB2312/UTF-8 编码兜底。

#### `async function 反代参数获取(request)` — [_worker.js:1697](file:///workspace/_worker.js)

从 URL 路径与查询参数提取代理配置。识别规则：

- `?proxyip=` 或 `/proxyip=` / `/proxyip.` / `/pyip=` / `/ip=` → 设置 `反代IP`，禁用兜底。
- `/socks5://` / `/http://`（带 `://`）→ 启用 SOCKS5/HTTP 全局反代。
- `/socks5=` / `/s5=` / `/gs5=` / `/http=` / `/ghttp=` → `g` 前缀启用全局。
- 支持 Base64 编码的用户名密码自动解码。

#### `async function 获取SOCKS5账号(address)` — [_worker.js:1757](file:///workspace/_worker.js)

解析 `user:pass@host:port` 形式的 SOCKS5 账号字符串，支持 IPv6（`[...]:port`）与 Base64 认证段。返回 `{ username, password, hostname, port }`。

#### `async function 解析地址端口(proxyIP, 目标域名, UUID)` — [_worker.js:1883](file:///workspace/_worker.js)

将 ProxyIP 解析为地址端口数组并缓存。支持三种形式：

- `*.william`：DoH 查询 TXT 记录，按 `\010` 或换行分割多个地址。
- 普通 IP/域名：直接解析；域名时并行 DoH 查询 A/AAAA 记录。
- `*.tp端口`：从域名后缀提取端口。

解析后按 `目标根域名 + UUID` 生成随机种子洗牌，取前 8 个，实现按目标站点稳定分布。

### 6.8 工具函数

#### `async function MD5MD5(文本)` — [_worker.js:1284](file:///workspace/_worker.js)

双重 MD5：先对原文 MD5，取十六进制的第 7~27 位再 MD5，返回 32 位小写十六进制。用于 cookie、token 等派生。

#### `function sha224(s)` — [_worker.js:1850](file:///workspace/_worker.js)

纯 JS 实现的 SHA-224（CF Workers 未原生提供），用于 Trojan 密码校验。

#### `function 随机路径(完整节点路径 = "/")` — [_worker.js:1298](file:///workspace/_worker.js)

从内置 ~200 个常用路径目录中随机选 1~3 个拼接，生成随机 WS 路径前缀，增强抗识别能力。

#### `async function 批量替换域名(内容, hosts, 每组数量 = 2)` — [_worker.js:1317](file:///workspace/_worker.js)

将订阅内容中的 `example.com` 占位符按 `每组数量` 个 host 一组进行批量替换，实现多域名负载。

#### `async function getECH(host)` — [_worker.js:1327](file:///workspace/_worker.js)

通过 DoH（`1.1.1.1`）查询 host 的 HTTPS 类型记录（type=65），解析 ECH 配置 base64。

#### `async function getCloudflareUsage(Email, GlobalAPIKey, AccountID, APIToken)` — [_worker.js:1792](file:///workspace/_worker.js)

调用 CF API 查询 Workers/Pages 当日请求用量。优先级：`APIToken` > `Email+GlobalAPIKey`；若无 AccountID 则先 `/accounts` 列表查找。通过 GraphQL 查询 `workersInvocationsAdaptive` 与 `pagesInvocationsAdaptive` 汇总。返回 `{ success, pages, workers, total, max }`。

#### `async function nginx()` / `async function html1101(host, 访问IP)` — [_worker.js:1996](file:///workspace/_worker.js) / [_worker.js:2026](file:///workspace/_worker.js)

返回内置 HTML 伪装页面字符串。`html1101` 模拟 CF Worker 1101 错误页，含 Ray ID 与 IP reveal 交互。

---

## 7. 配置数据结构（config_JSON）

`config_JSON` 由 `读取config_JSON` 生成，是贯穿订阅生成与后台管理的核心数据结构。默认值定义于 [_worker.js:1363-1426](file:///workspace/_worker.js)：

```javascript
{
  "TIME": "ISO 时间戳",
  "HOST": "当前 host",
  "HOSTS": ["hostname"],            // 多域名列表，用于订阅负载
  "UUID": "userID",
  "PATH": "/",                       // 节点路径
  "协议类型": "vless",               // vless 或 trojan
  "传输协议": "ws",
  "跳过证书验证": false,
  "启用0RTT": false,
  "TLS分片": null,                   // null | 'Shadowrocket' | 'Happ'
  "随机路径": false,
  "ECH": false,
  "ECHConfig": {
    "DNS": "https://doh.cmliussss.net/CMLiussss",
    "SNI": null
  },
  "Fingerprint": "chrome",
  "优选订阅生成": {
    "local": true,                   // true: 本地生成; false: 优选订阅生成器
    "本地IP库": {
      "随机IP": true,
      "随机数量": 16,
      "指定端口": -1
    },
    "SUB": null,                     // 外部优选订阅生成器 host
    "SUBNAME": "edgetunnel",         // 订阅文件名/节点备注
    "SUBUpdateTime": 3,              // 订阅更新间隔（小时）
    "TOKEN": "MD5MD5(host+userID)"
  },
  "订阅转换配置": {
    "SUBAPI": "https://SUBAPI.cmliussss.net",
    "SUBCONFIG": "https://raw.githubusercontent.com/cmliu/ACL4SSR/.../ACL4SSR_Online_Mini_MultiMode_CF.ini",
    "SUBEMOJI": false
  },
  "反代": {
    "PROXYIP": "auto",              // 'auto' 表示使用 colo.PrOxYIp.CmLiUsSsS.nEt
    "SOCKS5": {
      "启用": null,                 // 'socks5' | 'http' | 'https' | null
      "全局": false,
      "账号": "",
      "白名单": ["*tapecontent.net", ...]
    }
  },
  "TG": {
    "启用": false,
    "BotToken": null,               // 后台展示时掩码
    "ChatID": null
  },
  "CF": {
    "Email": null,
    "GlobalAPIKey": null,           // 掩码
    "AccountID": null,              // 掩码
    "APIToken": null,               // 掩码
    "UsageAPI": null,               // 第三方用量查询 URL
    "Usage": {
      "success": false,
      "pages": 0,
      "workers": 0,
      "total": 0,
      "max": 100000
    }
  },
  "完整节点路径": "/",               // 运行时计算
  "LINK": "vless://...",            // 运行时计算
  "加载时间": "XX.XXms"             // 运行时计算
}
```

---

## 8. KV 存储模型

Worker 通过环境变量 `env.KV` 绑定 Cloudflare KV 命名空间，存储以下键：

| Key | 类型 | 内容 | 写入方 |
|-----|------|------|--------|
| `config.json` | JSON | 主配置（[§7](#7-配置数据结构config_json)） | `读取config_JSON` / `POST /admin/config.json` |
| `cf.json` | JSON | CF API 凭证（Email/GlobalAPIKey/AccountID/APIToken/UsageAPI） | `POST /admin/cf.json` |
| `tg.json` | JSON | Telegram Bot 配置（BotToken/ChatID） | `POST /admin/tg.json` |
| `ADD.txt` | Text | 自定义优选 IP 列表（每行一个） | `POST /admin/ADD.txt` |
| `log.json` | JSON Array | 请求日志（FIFO，上限 4MB） | `请求日志记录` |

**注意**：`env.KV` 变量名固定不可修改（见 [wrangler.toml](file:///workspace/wrangler.toml) 注释）。未绑定 KV 且未设置 `env.UUID` 时，访问会返回 `/noKV` 提示页。

---

## 9. 环境变量与依赖关系

### 9.1 环境变量

| 变量名 | 必填 | 说明 |
|--------|:----:|------|
| `ADMIN` | ✅ | 后台管理密码；同时作为 userID 派生源 |
| `KEY` | ❌ | 快速订阅密钥；访问 `/{KEY}` 重定向到订阅；同时作为加密秘钥 |
| `UUID` | ❌ | 强制固定 UUID（UUIDv4）；未设置时由 `MD5MD5(ADMIN+KEY)` 派生 |
| `HOST` | ❌ | 强制固定伪装域名（支持多域名逗号分隔） |
| `PATH` | ❌ | 强制固定伪装路径 |
| `PROXYIP` | ❌ | 全局反代 IP（支持多 IP 逗号分隔随机） |
| `URL` | ❌ | 主页伪装地址（URL / `nginx` / `1101`） |
| `GO2SOCKS5` | ❌ | 强制走 SOCKS5 的域名白名单（`*` 全局，逗号分隔） |

### 9.2 绑定资源

- **KV Namespace**（变量名 `KV`）：必需，存储配置与日志。

### 9.3 外部服务依赖

| 服务 | 用途 | 调用点 |
|------|------|--------|
| `cloudflare:sockets` | TCP 连接（`connect`） | 协议转发、SOCKS5/HTTP 代理 |
| `1.1.1.1/dns-query` | DoH 解析（A/AAAA/TXT/HTTPS） | `解析地址端口` / `getECH` |
| `api.cloudflare.com/client/v4` | CF 用量查询 | `getCloudflareUsage` |
| `api.telegram.org` | TG 日志推送 | `sendMessage` |
| `edt-pages.github.io` | 后台/登录/提示静态页 | HTTP 路由各分支 |
| `speed.cloudflare.com/locations` | CF 测速点列表 | `/locations` |
| `raw.githubusercontent.com/cmliu/cmliu/main/CF-CIDR/*.txt` | CF 优选 CIDR 列表 | `生成随机IP` |
| `SUBAPI.cmliussss.net` | 订阅转换后端 | 订阅生成（非 mixed） |
| `doh.cmliussss.net/CMLiussss` | 默认 ECH DNS | `读取config_JSON` |
| `{colo}.PrOxYIp.CmLiUsSsS.nEt` | 默认 ProxyIP（按机房） | `fetch` 入口默认值 |
| `check.socks5.090227.xyz` | SOCKS5 可用性检测 | `SOCKS5可用性验证` |

### 9.4 模块间依赖关系

```
fetch 入口
  ├─► MD5MD5 ──► crypto.subtle
  ├─► 整理成数组
  ├─► 反代参数获取 ──► 获取SOCKS5账号
  ├─► 处理WS请求
  │     ├─► makeReadableStr ──► base64ToArray
  │     ├─► 解析木马请求 ──► sha224
  │     ├─► 解析魏烈思请求 ──► formatIdentifier
  │     ├─► forwardataTCP
  │     │     ├─► connectDirect ──► connect (cloudflare:sockets)
  │     │     ├─► connecttoPry
  │     │     │     ├─► socks5Connect / httpConnect
  │     │     │     └─► 解析地址端口 ──► DoH (1.1.1.1)
  │     │     └─► connectStreams
  │     └─► forwardataudp ──► connect (8.8.4.4:53)
  ├─► 读取config_JSON
  │     ├─► env.KV.get/put
  │     ├─► getCloudflareUsage ──► CF API
  │     ├─► 掩码敏感信息
  │     └─► MD5MD5
  ├─► 订阅生成 (/sub)
  │     ├─► 生成随机IP ──► fetch (CIDR 列表)
  │     ├─► 请求优选API ──► fetch (优选 API)
  │     ├─► Clash订阅配置文件热补丁
  │     ├─► Singbox订阅配置文件热补丁
  │     ├─► Surge订阅配置文件热补丁
  │     ├─► 随机路径 / 批量替换域名
  │     └─► getECH ──► DoH
  ├─► 请求日志记录 ──► sendMessage ──► TG API
  └─► nginx / html1101 / fetch(伪装页)
```

---

## 10. 订阅生成与协议适配

### 10.1 客户端类型识别

位于 [_worker.js:217-232](file:///workspace/_worker.js)，识别优先级：

1. SubConverter 请求（含 `b64`/`base64` 参数或特定 UA）→ `mixed`
2. `?target=` 参数 → 取该值
3. `?clash` 或 UA 含 `clash`/`meta`/`mihomo` → `clash`
4. `?sb`/`?singbox` 或 UA 含 `singbox`/`sing-box` → `singbox`
5. `?surge` 或 UA 含 `surge` → `surge&ver=4`
6. `?quanx` 或 UA 含 `quantumult` → `quanx`
7. `?loon` 或 UA 含 `loon` → `loon`
8. 兜底 → `mixed`

### 10.2 mixed 类型生成流程

1. 获取优选 IP 列表（本地 `ADD.txt` / 随机生成 / 优选订阅生成器）。
2. 对每个 IP/域名生成节点 LINK：`{协议}://00000000-...@{地址}:{端口}?security=tls&type=ws&host=example.com&fp=...&sni=example.com&path=...&encryption=none#{备注}`。
3. 占位 UUID `00000000-0000-4000-8000-000000000000` 与 `example.com` 后续被替换为真实 UUID 与 HOSTS。
4. 非 mozilla UA 或带 `b64` 参数时 Base64 编码输出。

### 10.3 非 mixed 类型生成流程

1. 构造订阅转换 URL：`{SUBAPI}/sub?target={type}&url={自身 mixed 订阅 URL}&config={SUBCONFIG}&emoji={SUBEMOJI}&scv={跳过证书验证}`。
2. `fetch` 获取转换后内容。
3. 按类型执行热补丁：
   - `clash` → `Clash订阅配置文件热补丁`（DNS/ECH 注入）
   - `singbox` → `Singbox订阅配置文件热补丁`（字段迁移/ECH）
   - `surge` → `Surge订阅配置文件热补丁`（ws 参数补全/MANAGED-CONFIG）

### 10.4 响应头

订阅响应包含特殊头：

- `Profile-Update-Interval`: 订阅更新间隔（小时）
- `Profile-web-page-url`: 后台地址
- `Subscription-Userinfo`: `upload=...; download=...; total=...; expire=4102329600`（2099-12-31）
- `Cache-Control: no-store`

---

## 11. 代理转发机制

### 11.1 转发优先级

`forwardataTCP` 中的转发决策（[_worker.js:520-599](file:///workspace/_worker.js)）：

```
if (启用SOCKS5反代 && (启用SOCKS5全局反代 || 命中白名单)):
    直接走 SOCKS5/HTTP 代理 (connecttoPry)
else:
    try:
        直连目标 (connectDirect)
        若无响应数据 → 回退 connecttoPry
    catch:
        走 connecttoPry
```

### 11.2 ProxyIP 反代逻辑

`connecttoPry` 在未启用 SOCKS5 时走 ProxyIP 分支：

1. `解析地址端口(反代IP, host, yourUUID)` 将 ProxyIP 解析为 `[地址, 端口][]` 数组（最多 8 个，按目标域名+UUID 洗牌）。
2. `connectDirect(atob('UFJPWFlJUC50cDEuMDkwMjI3Lnh5eg=='), 1, rawData, 所有反代数组, 启用反代兜底)` 轮询尝试连接反代数组，缓存命中索引。
3. 全部失败时根据 `启用反代兜底` 决定直连原目标或关闭 WS。

> 注：`atob('UFJPWFlJUC50cDEuMDkwMjI3Lnh5eg==')` 解码为 `PROXYIP.tp1.090227.xyz`，作为占位地址（实际连接使用 `所有反代数组`）。

### 11.3 SOCKS5 白名单

默认白名单（[_worker.js:6](file:///workspace/_worker.js)）：

```javascript
['*tapecontent.net', '*cloudatacdn.com', '*loadshare.org',
 '*cdn-centaurus.com', 'scholar.google.com']
```

可通过 `env.GO2SOCKS5` 覆盖。`*` 通配符通过 `new RegExp('^' + pattern.replace(/\*/g, '.*') + '$', 'i')` 匹配。

### 11.4 反代兜底

`启用反代兜底` 控制 ProxyIP 全部失败后的行为：

- `true`（默认，未设置 `env.PROXYIP` 时）：回退直连目标。
- `false`（设置了 `env.PROXYIP` 或路径指定 `proxyip=` 时）：关闭 WS，连接终止。

---

## 12. 项目运行方式

### 12.1 部署方式

支持三种部署方式（详见 [README.md](file:///workspace/README.md)）：

#### A. Cloudflare Workers 部署

1. CF Worker 控制台新建 Worker，粘贴 [_worker.js](file:///workspace/_worker.js) 内容。
2. `设置 → 变量` 添加 `ADMIN` = 管理员密码。
3. `绑定` 添加 KV 命名空间，变量名 `KV`。
4. `触发器 → 添加自定义域` 绑定域名。
5. 访问 `https://<域名>/admin` 登录。

#### B. Cloudflare Pages 上传部署（推荐）

1. 下载仓库 `main.zip`。
2. CF Pages `上传资产` 创建项目，上传 zip 部署。
3. `设置 → 环境变量` 添加 `ADMIN`。
4. 重新部署生效。
5. `设置 → 绑定` 添加 KV，变量名 `KV`。
6. `自定义域` 绑定 CNAME → `edgetunnel.pages.dev`。

#### C. Pages + GitHub 部署

1. Fork 仓库。
2. CF Pages `连接到 Git` 选择 `edgetunnel`。
3. 添加环境变量 `ADMIN`。
4. 绑定 KV（变量名 `KV`）。
5. 绑定自定义域。

> Fork 部署的仓库可通过 [.github/workflows/sync.yml](file:///workspace/.github/workflows/sync.yml) 每日自动同步上游。

### 12.2 本地开发（Wrangler）

```bash
# 安装 wrangler
npm install -g wrangler

# 登录
wrangler login

# 本地调试（需 wrangler.toml 配置 KV）
wrangler dev

# 部署
wrangler deploy
```

> 注：[wrangler.toml](file:///workspace/wrangler.toml) 中 KV 绑定默认被注释，需手动取消注释并填入 `id`，或通过 Pages 控制台绑定。

### 12.3 访问入口

| 路径 | 用途 | 鉴权 |
|------|------|------|
| `/login` | 登录页 | 无 |
| `/admin` | 后台管理面板 | Cookie `auth` |
| `/admin/config.json` | GET 读 / POST 存主配置 | Cookie |
| `/admin/cf.json` | GET request.cf / POST 存 CF 凭证 | Cookie |
| `/admin/tg.json` | POST 存 TG 配置 | Cookie |
| `/admin/ADD.txt` | GET 读 / POST 存优选 IP | Cookie |
| `/admin/log.json` | 读日志 | Cookie |
| `/admin/init` | 重置配置为默认 | Cookie |
| `/admin/check` | SOCKS5/HTTP 代理检测 | Cookie |
| `/admin/getADDAPI` | 验证优选 API | Cookie |
| `/admin/getCloudflareUsage` | 查询 CF 用量 | Cookie |
| `/{KEY}` | 快速订阅重定向 | KEY 正确 |
| `/sub?token=` | 订阅生成 | token 正确 |
| `/logout` | 清 cookie 跳登录 | 无 |
| `/locations` | CF 测速点列表 | Cookie |
| `/robots.txt` | Disallow | 无 |
| WS 升级请求 | VLESS/Trojan 代理 | 管理员密码非空 |

### 12.4 PATH 动态切换代理

运行时通过 URL PATH/查询参数动态指定代理方案（见 [README.md](file:///workspace/README.md) "高级实用技巧"）：

| 形式 | 含义 |
|------|------|
| `/proxyip=<ip>` 或 `?proxyip=<ip>` | 指定 ProxyIP |
| `/proxyip.<domain>` | 域名以 `proxyip.` 开头时指定反代 |
| `/socks5=user:pass@host:port` | 指定 SOCKS5（非全局） |
| `/socks5://user:pass@host:port` | 指定 SOCKS5（全局） |
| `/socks://<base64>@host:port` | Base64 认证，全局 SOCKS5 |
| `/http=user:pass@host:port` | 指定 HTTP 代理（非全局） |
| `/http://user:pass@host:port` | 指定 HTTP 代理（全局） |
| `/gs5=...` / `/ghttp=...` | `g` 前缀启用全局 |

---

## 13. 关键约束与注意事项

### 13.1 平台约束

- **单文件 ES Module**：入口必须 `export default { fetch }`，无 `package.json`。
- **`cloudflare:sockets`**：`connect` 仅在 Workers 运行时可用，本地 `wrangler dev` 需配置支持。
- **MD5 限制**：`crypto.subtle.digest` 在 Workers 中支持 MD5（用于 `MD5MD5`），但 sha224 需纯 JS 实现（`sha224` 函数）。
- **KV 容量**：单值上限 25MB，日志限 4MB（FIFO 滚动）。
- **CPU 时间**：Workers 免费版单请求 CPU 时间 10ms，付费版 50ms；订阅生成涉及多次外部 fetch，建议付费版。

### 13.2 安全约束

- `ADMIN` 密码必填，否则返回 `/noADMIN` 提示。
- Cookie 绑定 UA + 加密秘钥 + 密码，UA 变化需重新登录。
- CF/TG 凭证在后台展示时掩码，KV 中明文存储。
- `isSpeedTestSite` 屏蔽 `speed.cloudflare.com` 测速。
- 仅支持 TCP（cmd=1）与 UDP 53 端口 DNS（cmd=2, port=53），其他 UDP 拒绝。

### 13.3 兼容性

- `compatibility_date = "2025-11-04"`（[wrangler.toml](file:///workspace/wrangler.toml)）。
- Sing-box 热补丁处理 1.10.0+ 字段迁移。
- Clash 热补丁兼容 `mode: Rule` → `mode: rule`。
- Surge 热补丁补全 `ws=true` 等字段。

### 13.4 免责声明

项目仅供教育、科研及个人安全测试，使用者需遵守当地法律，作者不承担滥用责任（见 [README.md](file:///workspace/README.md) 末尾）。

---

## 附录：函数索引

| 函数名 | 行号 | 简述 |
|--------|------|------|
| `fetch` (入口) | 9 | 主入口，HTTP/WS 路由 |
| `处理WS请求` | 371 | WebSocket 升级与协议分发 |
| `解析木马请求` | 426 | Trojan 协议解析 |
| `解析魏烈思请求` | 485 | VLESS 协议解析 |
| `forwardataTCP` | 520 | TCP 转发（直连/反代/SOCKS5） |
| `forwardataudp` | 601 | UDP DNS 转发 |
| `closeSocketQuietly` | 628 | 安全关闭 socket |
| `formatIdentifier` | 636 | 字节数组转 UUID 格式 |
| `connectStreams` | 640 | 远端 readable → WS |
| `makeReadableStr` | 667 | WS → ReadableStream |
| `isSpeedTestSite` | 692 | 测速站点屏蔽 |
| `base64ToArray` | 706 | base64 → ArrayBuffer |
| `socks5Connect` | 720 | SOCKS5 客户端 |
| `httpConnect` | 756 | HTTP CONNECT 隧道 |
| `Clash订阅配置文件热补丁` | 789 | Clash YAML 后处理 |
| `Singbox订阅配置文件热补丁` | 1009 | Sing-box JSON 迁移 |
| `Surge订阅配置文件热补丁` | 1192 | Surge 订阅补全 |
| `请求日志记录` | 1211 | KV 日志写入 |
| `sendMessage` | 1244 | TG 消息推送 |
| `掩码敏感信息` | 1273 | 字符串掩码 |
| `MD5MD5` | 1284 | 双重 MD5 |
| `随机路径` | 1298 | 随机 WS 路径 |
| `随机替换通配符` | 1306 | 域名通配符替换 |
| `批量替换域名` | 1317 | 订阅多域名负载 |
| `getECH` | 1327 | DoH 查询 ECH 配置 |
| `读取config_JSON` | 1359 | 配置加载与合并 |
| `生成随机IP` | 1519 | CF 优选 IP 随机生成 |
| `整理成数组` | 1549 | 文本 → 数组 |
| `isValidBase64` | 1557 | Base64 校验 |
| `base64Decode` | 1571 | Base64 解码 |
| `请求优选API` | 1577 | 优选 API 并发请求 |
| `反代参数获取` | 1697 | URL 反代参数提取 |
| `获取SOCKS5账号` | 1757 | SOCKS5 账号解析 |
| `getCloudflareUsage` | 1792 | CF 用量查询 |
| `sha224` | 1850 | SHA-224 纯 JS 实现 |
| `解析地址端口` | 1883 | ProxyIP 解析与缓存 |
| `SOCKS5可用性验证` | 1971 | 代理可用性检测 |
| `nginx` | 1996 | nginx 欢迎页 HTML |
| `html1101` | 2026 | CF 1101 错误页 HTML |
