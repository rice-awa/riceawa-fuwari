
---
title: Minecraft Java 版服务端管理协议（MSMP）开发者使用文档
published: 2026-02-11
tags: ['Minecraft', 'MSMP', 'JSON-RPC', 'WebSocket', '服务端管理', '协议']
category: '教程'
draft: false
description: 详细介绍 Minecraft Java 版服务端管理协议（MSMP）的配置、认证、API 方法及最佳实践，帮助插件开发者快速上手使用这一官方原生管理接口。
---

# Minecraft Java 版服务端管理协议（MSMP）开发者使用文档

**文档版本**：2026-02-11  
**协议版本**：2.0.0 / 1.21.11+  
**目标读者**：插件开发者、服务器管理工具开发者、托管平台集成工程师

---

## 1. 协议概览与核心概念

### 1.1 协议定位与设计目标

Minecraft 服务端管理协议（Minecraft Server Management Protocol，简称 **MSMP**）是 Mojang 为 Minecraft Java 版专用服务器（Dedicated Server）官方引入的原生管理 API，首次亮相于 **2025 年 8 月 26 日发布的 25w35a 快照版本** 。该协议的诞生填补了 Minecraft 服务器管理生态中长期存在的标准化缺口，为插件开发者、服务器托管平台以及自动化运维工具提供了一套**标准化、实时化、双向交互**的服务器管理能力。

在 MSMP 出现之前，Minecraft 服务器管理长期依赖三类方案，各有显著局限。**RCON（Remote Console）协议**作为官方提供的远程管理手段，基于 TCP 明文传输，仅支持单向命令发送与文本响应，存在严重的安全与功能缺陷：通信内容未经加密，认证机制简陋（单一密码明文传输），且无法获取结构化数据或实时事件。**Query 协议**专注于服务器状态查询，但仅提供只读信息，缺乏管理操作能力，且无认证机制。**第三方 REST API 插件**（如 MinecraftServerAPI 、DeveloperMCP  等）虽在一定程度上弥补了功能缺陷，却引入了生态碎片化问题——实现质量参差不齐、与特定服务端软件深度耦合、版本兼容性风险高、缺乏标准化保证。

MSMP 从根本上解决了上述问题。作为**官方原生方案**，它直接集成于服务端核心（`server.jar`），无需额外插件即可启用；遵循语义化版本控制的 API 演进策略，确保向后兼容性；通过命名空间机制预留扩展空间，允许第三方在标准框架内添加自定义方法与事件 。协议采用 **WebSocket 作为传输层协议**，承载 **JSON-RPC 2.0** 格式的应用层消息，这一架构决策具有深远的工程意义：WebSocket 的全双工特性使得服务器能够在状态变更时主动向客户端推送通知，彻底改变了 RCON 时代"轮询查询"的低效模式；JSON-RPC 2.0 作为成熟的轻量级 RPC 规范，提供了清晰的请求-响应语义和标准化的错误处理机制，大幅降低了多语言客户端的开发门槛。

#### 1.1.1 官方原生管理接口与第三方方案对比

| 特性维度 | **MSMP** | RCON | Query | 第三方 REST API |
|---------|---------|------|-------|---------------|
| **传输协议** | WebSocket (RFC 6455) | 原始 TCP | UDP | HTTP/HTTPS |
| **应用协议** | JSON-RPC 2.0 | 自定义二进制 | 自定义二进制 | REST/JSON |
| **通信模式** | **双向全双工** | 单向请求-响应 | 单向请求-响应 | 单向请求-响应 |
| **实时通知** | ✅ **原生支持** | ❌ 不支持 | ❌ 不支持 | ⚠️ 需 Webhook/SSE |
| **认证机制** | **40位密钥 + TLS + Origin 验证** | 明文密码 | 无认证 | 自定义（API Key/JWT）|
| **加密传输** | ✅ **TLS 原生支持** | ❌ 不支持 | ❌ 不支持 | ⚠️ 依赖 HTTPS |
| **结构化数据** | ✅ **JSON 强类型** | 纯文本解析 | 有限字段 | 通常 JSON |
| **玩家管理** | ✅ **完整 CRUD** | ⚠️ 仅命令执行 | ❌ 仅查询 | 依赖插件实现 |
| **白名单管理** | ✅ **完整 CRUD** | ⚠️ 仅命令执行 | ❌ 不支持 | 依赖插件实现 |
| **管理员管理** | ✅ **完整 CRUD** | ⚠️ 仅命令执行 | ❌ 不支持 | 依赖插件实现 |
| **游戏规则修改** | ✅ **完整 CRUD** | ⚠️ 仅命令执行 | ❌ 不支持 | 依赖插件实现 |
| **服务端发现机制** | ✅ **`rpc.discover`** | ❌ 无 | ❌ 无 | 依赖插件文档 |
| **官方维护** | ✅ **是** | ✅ 是 | ✅ 是 | ❌ 否 |
| **版本同步更新** | ✅ **保证** | ✅ 保证 | ✅ 保证 | ⚠️ 不确定 |

MSMP 在**安全性、实时性、功能完整性**三个维度均实现了对传统协议的全面超越。特别是在双向通信和实时通知方面，MSMP 的 WebSocket 基础架构提供了原生支持，而 RCON 和 Query 作为早期设计的协议，其单向通信模式从根本上限制了实时管理能力。第三方 REST API 方案虽然可以通过 Webhook 或长轮询模拟实时效果，但实现复杂度高且可靠性差 。

#### 1.1.2 基于 WebSocket + JSON-RPC 2.0 的双向通信架构

MSMP 的通信架构由三个层次构成：**传输层的 WebSocket 连接**、**消息格式的 JSON-RPC 2.0 封装**，以及**应用层的命名空间方法调度**。WebSocket 连接建立后，客户端与服务端之间维持持久的全双工通道，所有管理操作与事件通知均在此通道上流转。相较于 HTTP 请求-响应模式，WebSocket 的长连接特性消除了频繁的 TCP 握手开销，将通信延迟降至最低；同时，服务端推送能力使得状态变更能够实时触达客户端，无需客户端主动轮询。

JSON-RPC 2.0 作为应用层协议，定义了两种消息类型：**请求（Request）**与**通知（Notification）**。请求消息包含唯一的 `id` 字段，服务端必须返回对应的响应（Response），形成完整的调用闭环；通知消息则无 `id` 字段，服务端不对其进行响应，适用于单向事件上报 。这一区分对于客户端实现至关重要：开发者需为方法调用维护请求-响应映射表（通常以 `id` 为键），而为通知注册独立的事件处理器。混合处理这两种消息类型是常见的初学者错误，会导致响应匹配失败或通知丢失。

命名空间（Namespace）是 MSMP 方法组织的核心机制。所有标准方法均归属于 `minecraft` 命名空间，格式为 `minecraft:<group>/<action>`，如 `minecraft:players`（获取玩家列表）、`minecraft:allowlist/add`（添加白名单）。通知事件则采用 `minecraft:notification/<event>` 格式，如 `minecraft:notification/players/joined`（玩家加入事件）。这种层级化的命名策略既保证了方法的可发现性，又为未来的功能扩展预留了清晰的路径。

#### 1.1.3 实时状态同步与事件驱动模型

MSMP 的事件驱动模型是其相较传统方案的核心竞争力。在 RCON 时代，获取玩家加入事件的唯一方式是定期执行 `list` 命令并比对输出差异——这种轮询模式不仅消耗服务器资源，还存在显著的延迟（取决于轮询间隔）。MSMP 通过 `minecraft:notification/players/joined` 和 `minecraft:notification/players/left` 通知，将事件延迟降至网络传输极限（通常 <10ms），且零服务器端轮询开销 。

事件驱动架构对插件开发范式产生了深远影响。以玩家管理插件为例，传统实现需维护定时任务扫描在线状态，而基于 MSMP 的实现则可完全事件化：注册通知处理器后，插件仅在事件触发时执行逻辑，资源占用与响应速度均获得数量级优化。更进一步，MSMP 的通知机制支持服务端状态的增量同步——当游戏规则（Gamerules）变更时，`minecraft:notification/game-rules/updated` 通知携带变更后的完整规则集，客户端无需重新查询即可更新本地缓存 。

### 1.2 版本演进与兼容性

#### 1.2.1 引入版本：25w35a / 1.21.9+

MSMP 于 **2025 年 8 月 26 日随 25w35a 快照首次发布** ，随后在 1.21.9 正式版中稳定化。这一版本节点对于插件开发者具有重要参考价值：任何目标版本低于 1.21.9 的服务端均不支持 MSMP，需在文档中明确标注兼容性要求。值得注意的是，MSMP 的版本演进遵循独立于 Minecraft 主版本的语义化版本控制——`rpc.discover` 方法返回的 API 版本信息（如 `2.0.0 (25w44a)`）反映了协议本身的迭代状态，而非底层游戏版本。

开发者在实现客户端库时，**应优先调用 `rpc.discover` 获取服务端支持的 API 版本与方法集合**，而非硬编码假设。这一防御性编程策略能够适配未来可能的方法增删或行为变更。例如，mc-rpc Rust 库即基于运行时发现动态生成方法绑定，确保与任意支持 MSMP 的服务端版本兼容。

#### 1.2.2 与 RCON、Query 等传统协议的功能对比

| 功能场景 | 推荐协议 | 理由 |
|---------|---------|------|
| **实时玩家监控面板** | **MSMP** | 事件通知替代轮询，延迟低、开销小 |
| **自动化备份脚本** | RCON | 需要执行 `/save-all` 等任意命令 |
| **服务器列表网站展示** | Query | 无认证查询，UDP 穿透性好 |
| **第三方管理面板** | **MSMP** | 结构化 API，功能完整，安全性高 |
| **紧急故障恢复** | RCON | 不依赖 MSMP 服务状态，底层可靠 |
| **跨版本兼容工具** | RCON/Query | 协议稳定，长期支持 |

MSMP、RCON 和 Query 三种协议在功能定位上形成明显的**互补关系**，而非简单的替代关系。理解它们的差异有助于在不同场景下选择最合适的工具 。

#### 1.2.3 未来扩展性：命名空间预留与自定义扩展机制

MSMP 的命名空间设计体现了前瞻性的扩展规划。除 `minecraft`（标准方法）和 `notification`（标准通知）两个保留命名空间外，协议明确允许第三方通过**自定义命名空间**添加方法与事件 。这一机制为服务端软件（Fabric、Forge、Paper 等）以及插件框架提供了标准化的扩展通道：Fabric API 可通过 `fabric:mods` 命名空间暴露模组列表查询，Paper 可通过 `paper:timings` 命名空间集成性能分析数据，而无需破坏与标准客户端的兼容性。

对于插件开发者，自定义命名空间的采用需遵循以下最佳实践：命名空间应采用**反向域名格式**（如 `com.example.myplugin`）以避免冲突；自定义方法应复用标准模式中的类型定义（如 `Player`、`Operator` 对象）以保持接口一致性；文档中需明确标注扩展的最低服务端版本与依赖条件。

### 1.3 核心术语表

#### 1.3.1 命名空间（Namespace）：`minecraft` 与 `notification`

命名空间是 MSMP 方法标识符的前缀段，用于逻辑分组与避免命名冲突。标准协议定义了两个保留命名空间：

| 命名空间 | 用途 | 格式示例 |
|---------|------|---------|
| **`minecraft`** | 承载所有管理操作方法 | `minecraft:<group>/<action>` |
| **`minecraft:notification/`** | 承载所有服务端主动推送的事件通知 | `minecraft:notification/<event>` |

`minecraft` 命名空间下的方法路径通常遵循 `group/action` 或 `group/subgroup/action` 的层级模式，例如 `minecraft:players`（获取玩家列表）、`minecraft:allowlist/add`（添加白名单）、`minecraft:server/save`（保存服务器）。这种层级设计使方法组织清晰，便于文档化和代码生成 。

`minecraft:notification/` 命名空间专用于服务端主动推送的事件通知。通知方法名同样采用层级结构，但语义侧重于状态变更而非操作指令，例如 `minecraft:notification/players/joined`（玩家加入）、`minecraft:notification/players/left`（玩家离开）、`minecraft:notification/game-rules/updated`（游戏规则更新）。通知与方法的命名空间分离，使客户端能够快速区分需要响应的请求和仅需处理的事件 。

#### 1.3.2 方法（Method）与通知（Notification）的区别

| 特征 | **方法（Method）** | **通知（Notification）** |
|-----|-------------------|------------------------|
| **消息方向** | 客户端 → 服务端 | 服务端 → 客户端 |
| **`id` 字段** | **必需**（用于响应匹配） | **必须省略** |
| **响应期望** | 服务端必须返回响应 | 无响应 |
| **典型用途** | 查询状态、执行操作 | 状态变更事件推送 |
| **错误处理** | 返回 `error` 对象 | 无（客户端本地处理） |
| **MSMP 示例** | `minecraft:players` | `notification:players/joined` |

方法与通知的区分是 JSON-RPC 2.0 规范的核心语义，MSMP 严格遵循这一区分。开发者在实现客户端时，需为方法调用维护请求-响应映射表（通常以 `id` 为键），而为通知注册独立的事件处理器。

#### 1.3.3 API 模式（Schema）与运行时发现机制

MSMP 提供两种 API 模式获取方式，内容镜像一致但适用场景各异：

| 方式 | 获取时机 | 主要用途 |
|-----|---------|---------|
| **`rpc.discover`** | 运行时 | 动态客户端适配，验证方法可用性 |
| **`json-rpc-api-schema.json`** | 构建时（数据生成器）| 静态代码生成，离线文档构建 |

模式文件采用 JSON Schema 风格描述，包含方法参数、返回值、通知载荷的结构定义。以 `operator` 类型为例 ：

```json
{
  "operator": {
    "properties": {
      "bypassesPlayerLimit": { "type": "boolean" },
      "permissionLevel": { "type": "integer" },
      "player": { "$ref": "#/components/schemas/player" }
    },
    "type": "object"
  }
}
```

运行时发现与静态生成的结合，使 MSMP 客户端能够在保证类型安全的同时，具备跨版本的适应能力。

---

## 2. 服务端配置与启用指南

### 2.1 基础配置项详解

MSMP 默认处于**禁用状态**，需通过 `server.properties` 文件显式启用。配置项的设计遵循"**最小权限原则**"与"**安全优先原则**"，所有涉及网络暴露的选项均提供细粒度的控制手段。

#### 2.1.1 `management-server-enabled`：启用开关

| 属性 | 值 |
|-----|---|
| 配置键 | `management-server-enabled` |
| 数据类型 | 布尔值（`true`/`false`） |
| 默认值 | `false` |
| 生效条件 | 服务端启动时读取 |

此为 MSMP 的总开关。设置为 `true` 后，服务端在启动时将初始化 WebSocket 服务器，监听配置的地址与端口 。若保持默认 `false`，则 MSMP 相关代码完全不加载，零运行时开销。生产环境建议仅在需要管理功能时启用，以减少攻击面。

#### 2.1.2 `management-server-host`：绑定地址（localhost/0.0.0.0）

| 属性 | 值 |
|-----|---|
| 配置键 | `management-server-host` |
| 数据类型 | 字符串（IP 地址或主机名） |
| 默认值 | `localhost` |
| 典型值 | `localhost`、`127.0.0.1`、`0.0.0.0`、具体公网 IP |

绑定地址决定了 WebSocket 服务器的网络可见范围。`localhost` 或 `127.0.0.1` 仅允许本机连接，适用于管理工具与服务器同机部署的场景；`0.0.0.0` 绑定所有网络接口，允许远程连接，但**必须配合 TLS 与严格认证**以确保安全 。云服务器部署时，建议显式指定内网 IP 而非 `0.0.0.0`，结合反向代理实现安全暴露。

#### 2.1.3 `management-server-port`：端口设置（默认随机/静态指定）

| 属性 | 值 |
|-----|---|
| 配置键 | `management-server-port` |
| 数据类型 | 整数（0-65535） |
| 默认值 | `0`（随机分配） |
| 典型值 | `25585`（官方示例）、`8080`、`8443` |

默认值为 `0` 时，服务端在启动时从操作系统获取可用临时端口，并将实际绑定端口写入日志 。这一设计简化了多实例部署（避免端口冲突），但增加了客户端配置的动态性。**静态端口配置**（如 `25585`）适用于需要防火墙规则预设或客户端硬编码的场景。端口选择应避开 Minecraft 游戏端口（默认 `25565`）及常用服务端口，减少扫描攻击风险。

#### 2.1.4 `management-server-secret`：40 位字母数字认证密钥

| 属性 | 值 |
|-----|---|
| 配置键 | `management-server-secret` |
| 数据类型 | 字符串（严格 40 字符） |
| 默认值 | 空（启动时自动生成） |
| 字符集 | `[A-Za-z0-9]` |

认证密钥是 MSMP 安全模型的核心。若配置为空，服务端在启动时**自动生成 40 位随机字符串**，并回写至 `server.properties` 文件 。自动生成的密钥强度足够，但会导致配置文件变更，需在版本控制中注意忽略或同步。生产环境建议显式配置密钥，便于多节点一致性与密钥轮换管理。

密钥的 40 位长度与字符集限制经过安全考量：总密钥空间为 `62^40 ≈ 2.7×10^71`，远超暴力破解可行性；字母数字排除易混淆字符（如 `O`/`0`、`I`/`l`），降低人工抄录错误。客户端提交密钥时，服务端进行严格格式验证，非 40 位字母数字的密钥立即拒绝，返回 401 错误。

### 2.2 安全配置与 TLS 加密

TLS 加密是 MSMP 生产环境的**强制要求**，协议设计将其作为默认安全策略。

#### 2.2.1 `management-server-tls-enabled`：TLS 开关

| 属性 | 值 |
|-----|---|
| 配置键 | `management-server-tls-enabled` |
| 数据类型 | 布尔值 |
| 默认值 | `true` |

默认 `true` 体现了**默认安全（Secure by Default）**的设计原则。启用后，WebSocket 端点从 `ws://` 升级为 `wss://`，所有通信内容经 TLS 1.2/1.3 加密，防止中间人攻击与窃听 。

#### 2.2.2 `management-server-tls-keystore`：PKCS12 证书库路径

| 属性 | 值 |
|-----|---|
| 配置键 | `management-server-tls-keystore` |
| 数据类型 | 字符串（文件路径） |
| 格式要求 | **PKCS #12**（`.p12` 或 `.pfx`）|

Java 生态的标准 TLS 配置方式，指向 PKCS#12 格式的密钥库文件。密钥库包含服务器证书与私钥，可由 Let's Encrypt 等 CA 签发，或自签名生成 。

#### 2.2.3 `management-server-tls-keystore-password`：密钥库密码（多来源优先级）

密钥库访问密码的配置需考虑安全性与便利性的平衡。Java 支持**三层密码来源优先级**，从高到低：

| 优先级 | 来源 | 适用场景 |
|-------|------|---------|
| 1 | 环境变量 `MINECRAFT_MANAGEMENT_TLS_KEYSTORE_PASSWORD` | **推荐**，容器化与自动化部署 |
| 2 | JVM 系统属性 `-Dmanagement.tls.keystore.password=` | 多实例隔离 |
| 3 | `server.properties` 配置项 | 开发与测试环境 |

生产环境**强烈建议采用环境变量注入**，避免密码泄露于配置文件或版本控制 。

#### 2.2.4 自签名证书生成与浏览器信任配置

开发测试场景下，自签名证书是便捷选择。使用 OpenSSL 生成：

```bash
# 生成私钥与证书
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
  -sha256 -days 365 -nodes -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

# 转换为 PKCS12
openssl pkcs12 -export -out minecraft.p12 -inkey key.pem -in cert.pem \
  -name minecraft -password pass:changeit
```

浏览器环境连接自签名 WSS 端点时，需先将证书导入系统信任库，或访问 `https://localhost:<port>` 手动确认例外（WebSocket 无法直接处理证书警告）。Node.js 等环境可通过 `NODE_TLS_REJECT_UNAUTHORIZED=0` 临时禁用验证（**仅限开发使用**）。

### 2.3 跨域与来源控制

#### 2.3.1 `management-server-allowed-origins`：浏览器 WebSocket 来源白名单

| 属性 | 值 |
|-----|---|
| 配置键 | `management-server-allowed-origins` |
| 数据类型 | 逗号分隔的字符串列表 |
| 默认值 | 空字符串（**拒绝所有浏览器连接**）|
| 示例值 | `https://admin.example.com,https://panel.example.org` |

`allowed-origins` 是 MSMP 针对浏览器环境的安全强化机制。WebSocket 连接建立时，浏览器自动附加 `Origin` HTTP 头，服务端验证该值是否在白名单内 。验证失败时，立即返回 HTTP 401 Unauthorized，终止连接。

这一机制有效防御了**跨站 WebSocket 劫持（CSWSH）攻击**：即使攻击者获取了有效密钥，若其网页来源不在白名单，浏览器将阻止 WebSocket 连接建立。配置时需注意：`Origin` 值包含协议与端口（如 `https://admin.example.com:8443`），白名单条目需精确匹配。

#### 2.3.2 Origin 验证失败时的 401 响应行为

来源验证失败与认证密钥失败的响应码均为 401，但日志中应区分具体原因以便排查。典型的错误响应结构：

```http
HTTP/1.1 401 Unauthorized
Content-Type: text/plain

Origin not allowed: https://evil.com
```

或

```http
HTTP/1.1 401 Unauthorized
Content-Type: text/plain

Invalid or missing authentication
```

生产环境建议启用详细日志记录，监控异常来源的访问尝试，作为安全态势感知的数据源。

### 2.4 启动验证与日志诊断

#### 2.4.1 成功启动标志：`Starting json RPC server on ...`

服务端成功初始化 MSMP 时，日志输出关键信息：

```
[Server thread/INFO]: Starting json RPC server on localhost/127.0.0.1:25585
[Server thread/INFO]: Json-RPC Management connection listening on localhost/127.0.0.1:25585
```

首行确认绑定地址与端口，第二行确认服务器已进入监听状态。若端口配置为 0，此处显示实际分配的动态端口。两行日志的线程标识均为 `Server thread`，表明 MSMP 与主游戏服务器共享线程——**MSMP 处理函数应避免阻塞操作**，以免影响游戏 tick 。

#### 2.4.2 绑定失败处理与端口冲突排查

| 错误场景 | 日志特征 | 解决方案 |
|---------|---------|---------|
| 端口被占用 | `Address already in use: bind` | 更换端口，或查找并终止占用进程 |
| 端口无效 | 端口范围错误提示 | 检查端口范围 1-65535 |
| 绑定地址无效 | `Cannot assign requested address` | 确认 IP 已配置在网络接口 |
| 权限不足 | `Permission denied` | 使用高位端口，或以 root 运行（不推荐）|
| TLS 密钥库错误 | `Keystore was tampered with, or password was incorrect` | 验证密码，检查文件完整性 |

端口冲突排查命令（Linux）：
```bash
sudo ss -tlnp | grep :8080  # 查看占用端口的进程
sudo lsof -i :8080          # 替代命令
```

#### 2.4.3 防火墙与云服务器安全组配置

| 场景 | 配置要求 |
|-----|---------|
| 本地开发 | 无需额外配置 |
| 同机管理工具 | 确保 `localhost` 可达 |
| 远程管理（内网）| 防火墙放行指定端口，绑定内网 IP |
| 远程管理（公网）| **安全组放行 + TLS 强制 + 来源白名单** |

云服务器（AWS、Azure、阿里云等）需在安全组/网络 ACL 中显式放行 MSMP 端口。建议**限制源 IP 范围至管理节点**，而非 `0.0.0.0/0`，形成纵深防御 。

---

## 3. 连接建立与认证机制

### 3.1 WebSocket 连接端点

#### 3.1.1 非 TLS 端点：`ws://<host>:<port>`

开发测试环境的默认连接方式。URL 格式示例：

```
ws://localhost:25585
ws://192.168.1.100:8080
```

**非 TLS 连接的所有内容（包括认证密钥）以明文传输，仅适用于可信网络环境**。生产环境公网暴露必须启用 TLS 。

#### 3.1.2 TLS 端点：`wss://<host>:<port>`

生产环境的标准连接方式。URL 格式示例：

```
wss://mc.example.com:8443
wss://10.0.0.5:25585
```

TLS 握手在 TCP 连接建立后立即开始，客户端验证服务端证书的可信性。证书验证失败（自签名、过期、主机名不匹配）会导致连接终止，错误信息因客户端而异 。

### 3.2 认证方式（二选一）

MSMP 提供两种**互斥的认证机制**，分别适用于不同客户端环境 ：

#### 3.2.1 `Authorization` 请求头：`Bearer <40位密钥>`

标准 HTTP 认证头格式，适用于**非浏览器客户端**（服务器端应用、命令行工具、桌面程序）：

```javascript
// Node.js 示例
const ws = new WebSocket('ws://localhost:25585', {
  headers: {
    'Authorization': 'Bearer AbCdEfGhIjKlMnOpQrStUvWxYz1234567890AbCd'
  }
});
```

```python
# Python websockets 示例
import websockets

async with websockets.connect(
    'ws://localhost:25585',
    extra_headers={'Authorization': 'Bearer ' + SECRET}
):
    ...
```

#### 3.2.2 `Sec-WebSocket-Protocol` 请求头：`minecraft-v1,<40位密钥>`

专门为**浏览器 WebSocket API** 设计的认证方式。浏览器 WebSocket 构造函数不支持自定义请求头，但原生支持 `protocols` 参数，该参数值会编码为 `Sec-WebSocket-Protocol` 头发送 ：

```javascript
// 浏览器原生 WebSocket API
const ws = new WebSocket(
  'wss://mc.example.com:8443',
  ['minecraft-v1', 'AbCdEfGhIjKlMnOpQrStUvWxYz1234567890AbCd']
);
```

`Sec-WebSocket-Protocol` 头的标准格式为逗号分隔的子协议列表，MSMP 约定首项为固定字符串 `minecraft-v1`，第二项为认证密钥。服务端解析时验证这一结构，提取密钥进行认证。

#### 3.2.3 浏览器环境强制使用 `Sec-WebSocket-Protocol` 的原因

浏览器安全模型（CORS、CSP）严格限制 WebSocket 连接的可控性，禁止脚本设置任意 HTTP 头，以防止请求伪造攻击。`Sec-WebSocket-Protocol` 是少数可由 JavaScript 控制的头部之一，因其属于 WebSocket 标准协议的有机组成部分。MSMP 的这一设计妥协，确保了**纯浏览器端管理面板**（如基于 React/Vue 的 Web UI）的可行性，无需后端代理即可直接连接服务端 。

### 3.3 认证失败处理

#### 3.3.1 HTTP 401 Unauthorized 响应

认证失败时，WebSocket 握手阶段即返回 HTTP 401，连接未升级即终止。客户端表现为 `WebSocket` 构造抛出错误或连接事件触发失败。错误响应体包含具体原因，但浏览器端因安全限制通常无法读取，需服务端日志配合排查 。

常见 401 原因：
- 完全缺失认证头
- `Authorization` 头格式错误（非 `Bearer <token>`）
- `Sec-WebSocket-Protocol` 头格式错误（缺少 `minecraft-v1` 或令牌）
- 令牌与 `management-server-secret` 不匹配
- 浏览器客户端的来源不在 `allowed-origins` 中

#### 3.3.2 密钥格式验证：严格 40 位字母数字

服务端对密钥进行两层验证：长度严格等于 40 字符，字符集限定于 `[A-Za-z0-9]`。任何偏离均立即 401 拒绝，不进入后续处理。这一严格验证防止了编码错误、截断密钥等常见配置问题的安全降级。

### 3.4 连接示例代码

#### 3.4.1 命令行工具：websocat 连接演示

[websocat](https://github.com/vi/websocat) 是功能强大的命令行 WebSocket 工具，适用于快速测试与脚本集成 ：

```bash
# 无认证尝试（预期失败）
$ websocat ws://localhost:40745
websocat: WebSocketError: Received unexpected status code (401 Unauthorized)

# 正确认证连接
$ websocat ws://localhost:40745 -H 'Authorization: Bearer n2pQcIG1OQ92jot2xG1M0aw0ZWnrh4F3Z3jw8qRP'
{"jsonrpc":"2.0","method":"server/status","id":0}
{"jsonrpc":"2.0","id":0,"result":{"started":true,"version":{"name":"25w37a","protocol":1073742092}}}

# TLS 连接（自签名证书）
$ websocat wss://localhost:40745 -H 'Authorization: Bearer ...' --insecure
# 或指定 CA 证书
$ websocat wss://localhost:40745 -H 'Authorization: Bearer ...' --ca-file cert.pem
```

#### 3.4.2 Node.js 客户端：mc-server-management 库用法

Aternos 团队维护的 `mc-server-management` 是目前最成熟的 Node.js 客户端库，提供高层 API 封装 ：

```javascript
const { Server } = require('mc-server-management');

const server = new Server('ws://localhost:25585', {
  secret: 'AbCdEfGhIjKlMnOpQrStUvWxYz1234567890AbCd'
});

// 获取操作员列表包装器
const ops = server.operatorList();

// 异步获取所有操作员（自动缓存，通过通知更新）
const operators = await ops.get();
console.log(operators);

// 添加操作员（多种参数形式）
await ops.add('Notch');                          // 名称，默认等级
await ops.add('jeb_', 4, true);                  // 名称、等级4、绕过限制
await ops.add(Player.withId('uuid-here'), 2);    // UUID 对象

// 批量操作
await ops.add(['player1', 'player2'], 3, true);
await ops.remove(['player1', 'player2']);

// 设置完整列表（原子替换）
await ops.set([
  new Operator('Notch', 4, true),
  'jeb_',  // 使用默认参数
  Player.withId('uuid')
], 3, true);

// 清空列表
await ops.clear();
```

该库的核心价值在于：**自动处理连接管理、请求序列化、响应解析；基于通知实现本地缓存的增量更新；提供类型友好的批量操作接口**。

#### 3.4.3 浏览器原生 WebSocket API 连接

```html
<!DOCTYPE html>
<html>
<head>
  <title>MSMP 管理面板</title>
</head>
<body>
  <pre id="output"></pre>
  <script>
    const SECRET = 'AbCdEfGhIjKlMnOpQrStUvWxYz1234567890AbCd';
    const ws = new WebSocket(
      'ws://localhost:25585',
      ['minecraft-v1', SECRET]
    );

    ws.onopen = () => {
      log('Connected');
      // 获取玩家列表
      send({ jsonrpc: '2.0', method: 'minecraft:players', id: 1 });
    };

    ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      log('Received: ' + JSON.stringify(msg, null, 2));
    };

    ws.onerror = (err) => log('Error: ' + err);
    ws.onclose = () => log('Disconnected');

    function send(msg) {
      ws.send(JSON.stringify(msg));
    }
    function log(text) {
      document.getElementById('output').textContent += text + '\n';
    }
  </script>
</body>
</html>
```

浏览器示例展示了 `Sec-WebSocket-Protocol` 认证、消息收发、事件处理的基础模式。生产环境应添加**重连逻辑、错误提示、UI 状态管理**等增强功能。

---

## 4. JSON-RPC 2.0 通信规范

### 4.1 请求格式

#### 4.1.1 必需字段：`jsonrpc`、`method`、`id`

| 字段 | 类型 | 必需 | 说明 |
|-----|------|------|------|
| `jsonrpc` | 字符串 | 是 | 固定值 `"2.0"`，标识协议版本 |
| `method` | 字符串 | 是 | 完整命名空间方法名 |
| `id` | 字符串/数字/null | 是 | 请求标识，用于响应匹配 |
| `params` | 数组/对象 | 否 | 方法参数，**MSMP 采用数组包裹** |

`id` 字段的选型建议：**数字递增**简单高效；**字符串 UUID** 避免冲突；`null` 仅用于通知（但通知应**省略 `id`** 而非设为 `null`）。客户端应确保同一连接内 `id` 的唯一性，通常采用单调递增计数器。

#### 4.1.2 参数传递：`params` 数组包裹约定

MSMP 的方法参数**统一采用数组包裹**，即使单参数也置于数组内 ：

```json
// ✅ 正确：单参数数组包裹
{"method":"minecraft:allowlist/add","id":1,"params":[[{"name":"jeb_"}]]}

// ❌ 错误：直接传递对象
{"method":"minecraft:allowlist/add","id":1,"params":{"name":"jeb_"}}
```

这一设计源于 JSON-RPC 2.0 的 `params` 可为数组或对象的灵活性，MSMP 统一为数组以简化服务端解析逻辑。**双层数组 `[[...]]`** 在批量操作场景中出现：外层数组为参数列表（本方法仅一个参数），内层数组为批量元素数组。

#### 4.1.3 命名空间方法完整格式：`minecraft:<group>/<action>`

方法字符串的解析规则：以冒号分隔命名空间与路径，路径中以斜杠分隔功能组与操作。当前版本的标准功能组包括：

| 功能组 | 操作示例 | 说明 |
|-------|---------|------|
| `players` | `players` | 获取在线玩家列表 |
| `allowlist` | `allowlist`, `allowlist/add`, `allowlist/remove` | 白名单查询与修改 |
| `operators` | `operators`, `operators/add`, `operators/remove` | 管理员查询与修改 |
| `server` | `server/save`, `server/stop`, `server/status` | 服务器控制 |
| `server-settings` | `server-settings`, `server-settings/set` | 服务器设置 |
| `game-rules` | `game-rules`, `game-rules/set` | 游戏规则 |

### 4.2 响应格式

#### 4.2.1 成功响应：`result` 字段结构

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": [
    {"id": "853c80ef-3c37-49fd-aa49-938b674adae6", "name": "jeb_"}
  ]
}
```

成功响应保留请求的 `id`，`result` 字段内容为方法特定的返回值。数组、对象、标量均可能，具体以 `rpc.discover` 模式或本文档方法章节为准。

#### 4.2.2 错误响应：`error` 对象（code、message、data）

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32601,
    "message": "Method not found",
    "data": "Method not found: minecraft:foo/bar"
  }
}
```

| 字段 | 类型 | 说明 |
|-----|------|------|
| `code` | 整数 | 标准或应用定义的错误码 |
| `message` | 字符串 | 人类可读的错误描述 |
| `data` | 任意 | 附加诊断信息，可选 |

### 4.3 标准错误码对照

#### 4.3.1 `-32601` Method not found：方法不存在

`-32601` 是 JSON-RPC 2.0 标准错误码，表示请求的方法未被服务端识别 。MSMP 场景下的典型原因：

- 命名空间拼写错误（如 `minecraft:player` 而非 `minecraft:players`）
- 服务端版本不支持该方法（如旧版本无 `server-settings` 组）
- 方法尚未实现（自定义命名空间的预期行为）

客户端应将此错误视为**致命或降级信号**：致命意味着服务端版本不兼容，需升级或切换方法；降级意味着功能不可用，可提示用户或禁用相关 UI。

#### 4.3.2 JSON-RPC 2.0 规范错误码全集

| 错误码 | 名称 | 含义 |
|--------|------|------|
| `-32700` | Parse error | 服务端接收到的 JSON 无法解析 |
| `-32600` | Invalid Request | 请求对象结构无效 |
| `-32601` | Method not found | 请求的方法不存在 |
| `-32602` | Invalid params | 方法参数无效（类型、数量、值）|
| `-32603` | Internal error | 服务端内部错误 |
| `-32000` 至 `-32099` | Server error | 预留的应用定义错误 |

MSMP 可能扩展 `-32000` 范围的错误码表示特定领域错误（如玩家不存在、权限不足），客户端应预留处理未知错误码的弹性。

### 4.4 通知消息处理

#### 4.4.1 服务端主动推送特征：无 `id` 字段

```json
{
  "jsonrpc": "2.0",
  "method": "minecraft:notification/players/joined",
  "params": [{"id": "853c80ef-3c37-49fd-aa49-938b674adae6", "name": "jeb_"}]
}
```

通知消息的关键识别特征：**`id` 字段完全缺失**（非 `null`），`method` 字段以 `notification:` 开头。客户端解析时应**优先检查 `id` 存在性**，区分响应与通知的处理路径。

#### 4.4.2 通知命名空间：`notification:<event>`

| 事件类别 | 事件示例 | 触发时机 |
|---------|---------|---------|
| 玩家生命周期 | `minecraft:notification/players/joined`, `minecraft:notification/players/left` | 玩家加入/离开服务器 |
| 游戏规则 | `minecraft:notification/game-rules/updated` | 任意游戏规则被修改 |

通知的 `params` 载荷结构与对应查询方法的返回元素一致，便于客户端复用解析逻辑。

---

## 5. 核心 API 方法组详解

### 5.1 运行时 API 发现

#### 5.1.1 `rpc.discover`：动态获取服务端支持的方法与通知

`rpc.discover` 是 MSMP 的元方法，无需认证即可调用（或作为首个验证连接的探针），返回当前服务端完整的 API 能力描述 ：

```json
// 请求
{"jsonrpc":"2.0","method":"rpc.discover","id":1}

// 响应（结构示意）
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "jsonrpc": "2.0",
    "info": {
      "title": "Minecraft Server JSON-RPC",
      "version": "2.0.0 (25w44a)"
    },
    "methods": [
      {
        "name": "minecraft:players",
        "params": [],
        "result": {"$ref": "#/components/schemas/PlayerArray"}
      }
      // ... 更多方法
    ],
    "notifications": [
      {
        "name": "notification:players/joined",
        "params": [{"$ref": "#/components/schemas/Player"}]
      }
      // ... 更多通知
    ],
    "components": {
      "schemas": {
        "Player": {
          "type": "object",
          "properties": {
            "id": {"type": "string", "format": "uuid"},
            "name": {"type": "string"}
          }
        }
        // ... 更多类型
      }
    }
  }
}
```

#### 5.1.2 响应结构：方法列表、参数模式、通知事件

| 字段 | 内容 |
|-----|------|
| `info` | API 元数据（标题、版本）|
| `methods` | 支持的方法数组，每项含 `name`、`params`（参数模式）、`result`（返回模式）|
| `notifications` | 支持的通知数组，每项含 `name`、`params`（事件载荷模式）|
| `components/schemas` | 可复用的类型定义，以 JSON Schema 格式描述 |

#### 5.1.3 与静态 `json-rpc-api-schema.json` 的关系

| 特性 | `rpc.discover` | `json-rpc-api-schema.json` |
|-----|---------------|---------------------------|
| 获取时机 | 运行时 | 构建时（数据生成器）|
| 内容时效 | 反映当前服务端实际能力 | 反映构建时的 API 版本 |
| 主要用途 | 动态客户端适配 | 静态代码生成 |
| 依赖条件 | 需运行服务端 | 需执行数据生成器 |

mc-rpc Rust 库的构建流程展示了静态模式的典型用法：`build.rs` 在编译期解析 `json-rpc-api-schema.json`，为每个方法生成异步函数，为每个类型生成 Rust 结构体，实现零开销的类型安全 。

### 5.2 玩家管理（`minecraft:players`）

#### 5.2.1 `minecraft:players`：获取在线玩家列表

```json
// 请求
{"jsonrpc":"2.0","method":"minecraft:players","id":1}

// 响应
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": [
    {"id": "853c80ef-3c37-49fd-aa49-938b674adae6", "name": "jeb_"},
    {"id": "069a79f4-44e9-4726-a5be-fca90e38aaf5", "name": "Notch"}
  ]
}
```

返回当前在线玩家的完整列表。空服务器返回空数组 `[]`。

#### 5.2.2 玩家对象结构：UUID（`id`）、名称（`name`）

| 字段 | 类型 | 格式 | 说明 |
|-----|------|------|------|
| `id` | 字符串 | UUID v4（带连字符）| 玩家唯一标识，离线模式为离线 UUID |
| `name` | 字符串 | 3-16 字符，合法 Minecraft 用户名 | 玩家显示名称 |

玩家对象是 MSMP 的核心引用类型，广泛嵌套于管理员、白名单等结构中。UUID 的解析与验证需注意：离线服务器（`online-mode=false`）使用基于用户名的离线 UUID 生成算法，与正版 UUID 不同，但格式一致。

#### 5.2.3 相关通知：`minecraft:notification/players/joined`、`minecraft:notification/players/left`

```json
// 玩家加入通知
{
  "jsonrpc": "2.0",
  "method": "minecraft:notification/players/joined",
  "params": [{"id": "...", "name": "NewPlayer"}]
}

// 玩家离开通知
{
  "jsonrpc": "2.0",
  "method": "minecraft:notification/players/left",
  "params": [{"id": "...", "name": "LeavingPlayer"}]
}
```

通知载荷为**单个玩家对象**，而非数组。客户端可通过维护本地在线玩家集合，响应通知进行增量更新，避免频繁全量查询。

### 5.3 白名单管理（`minecraft:allowlist`）

| 方法 | 功能 | 参数特点 |
|------|------|---------|
| `minecraft:allowlist` | 查询白名单列表 | 无参数 |
| `minecraft:allowlist/add` | 添加白名单条目 | **支持批量**，参数为玩家对象数组 |
| `minecraft:allowlist/remove` | 移除白名单条目 | **支持批量** |

#### 5.3.2 `minecraft:allowlist/add`：添加白名单（支持 UUID/名称）

```json
// 请求：通过名称添加
{"method":"minecraft:allowlist/add","id":1,"params":[[{"name":"jeb_"}]]}

// 请求：通过 UUID 添加
{"method":"minecraft:allowlist/add","id":1,"params":[[{"id":"853c80ef-3c37-49fd-aa49-938b674adae6"}]]}

// 请求：批量添加
{"method":"minecraft:allowlist/add","id":1,"params":[[
  {"name":"player1"},
  {"id":"uuid-2"},
  {"name":"player3"}
]]}

// 响应：成功添加的玩家列表（含解析后的 UUID）
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": [
    {"id": "...", "name": "jeb_"}
  ]
}
```

`params` 的双层数组结构：外层数组为方法参数列表（本方法仅一个参数），内层数组为批量操作条目。每个条目为对象，含 `name` 或 `id` 字段。

#### 5.3.4 批量操作与单条操作的参数差异

| 操作类型 | 参数结构 | 示例 |
|---------|---------|------|
| 单条添加 | `[[{"name":"player"}]]` | 外层数组1元素，内层数组1元素 |
| 批量添加 | `[[{"name":"p1"},{"id":"uuid"}]]` | 外层数组1元素，内层数组N元素 |

批量操作的设计体现了性能优化考量：单次网络往返处理多个条目，减少 WebSocket 帧传输开销。

### 5.4 管理员管理（`minecraft:operators`）

| 方法 | 功能 |
|------|------|
| `minecraft:operators` | 查询管理员列表 |
| `minecraft:operators/add` | 添加管理员，**支持等级和绕过设置** |
| `minecraft:operators/remove` | 移除管理员 |

#### 5.4.2 `minecraft:operators/add`：添加管理员（支持等级与绕过限制）

添加管理员时，可以指定两个额外参数：
- **`level`**：权限等级，1-4，控制可用命令范围
- **`bypass`**：布尔值，是否绕过玩家数量限制

`mc-server-management` 库提供了优雅的操作员对象抽象 ：

```javascript
// 使用默认等级和绕过设置
await ops.add('Aternos');

// 显式指定等级4，绕过限制
await ops.add('Aternos', 4, true);

// 通过 UUID 添加
await ops.add(Player.withId('player-uuid'), 2, false);
```

### 5.5 服务器控制（`minecraft:server`）

| 方法 | 功能 | 注意事项 |
|------|------|---------|
| `minecraft:server/save` | 保存世界数据 | 可能耗时较长，大服务器需考虑异步处理 |
| `minecraft:server/stop` | **优雅关闭服务器** | 触发保存并断开所有玩家 |
| `minecraft:server/status` | 查询服务器状态 | 包含启动状态和版本信息 |

状态查询响应示例 ：

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "result": {
    "started": true,
    "version": {
      "name": "25w37a",
      "protocol": 1073742092
    }
  }
}
```

### 5.6 服务器设置（`minecraft:server-settings`）

服务器设置管理允许查询和修改 `server.properties` 中的配置项。修改后的行为取决于具体设置——某些变更需要重启生效，某些则可以热加载。协议层面不区分这两种情况，客户端应通过文档或实验确定具体行为。

### 5.7 游戏规则（`minecraft:game-rules`）

| 方法 | 功能 |
|------|------|
| `minecraft:game-rules` | 查询所有游戏规则 |
| `minecraft:game-rules/set` | 修改指定游戏规则 |

修改后会触发通知：

- **`minecraft:notification/game-rules/updated`**：游戏规则发生变更

### 5.8 封禁玩家列表管理（`minecraft:bans`）

封禁玩家列表相关方法共有5个，分别用于获取、设置、添加、移除及清空封禁玩家列表。

| 方法 | 功能 | 参数 |
|------|------|------|
| `minecraft:bans` | 获取当前封禁玩家列表 | 无 |
| `minecraft:bans/set` | 用新的封禁列表覆盖当前列表 | 封禁条目数组 |
| `minecraft:bans/add` | 将玩家添加到封禁列表中 | 封禁条目数组 |
| `minecraft:bans/remove` | 将玩家移出封禁列表 | 玩家对象数组 |
| `minecraft:bans/clear` | 清空当前封禁列表 | 无 |

#### 封禁条目结构

```json
{
  "player": {"id": "uuid", "name": "playerName"},
  "reason": "封禁原因",
  "source": "操作者",
  "expires": "2026-01-01T00:00:00Z"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `player` | 对象 | 被封禁的玩家（PlayerDto）|
| `reason` | 字符串 | 封禁原因 |
| `source` | 字符串 | 执行封禁的操作者 |
| `expires` | 字符串 | 封禁过期时间（ISO 8601格式），空字符串表示永久封禁 |

### 5.9 封禁 IP 列表管理（`minecraft:ip_bans`）

封禁 IP 列表相关方法共有5个，分别用于获取、设置、添加、移除及清空封禁 IP 列表。

| 方法 | 功能 | 参数 |
|------|------|------|
| `minecraft:ip_bans` | 获取当前封禁 IP 列表 | 无 |
| `minecraft:ip_bans/set` | 用新的封禁列表覆盖当前列表 | IP 封禁条目数组 |
| `minecraft:ip_bans/add` | 将 IP 添加到封禁列表中 | IP 封禁条目数组 |
| `minecraft:ip_bans/remove` | 将 IP 移出封禁列表 | IP 地址字符串数组 |
| `minecraft:ip_bans/clear` | 清空当前封禁 IP 列表 | 无 |

#### IP 封禁条目结构

```json
{
  "ip": "192.168.1.1",
  "reason": "封禁原因",
  "source": "操作者",
  "expires": "2026-01-01T00:00:00Z"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `ip` | 字符串 | 被封禁的 IP 地址 |
| `reason` | 字符串 | 封禁原因 |
| `source` | 字符串 | 执行封禁的操作者 |
| `expires` | 字符串 | 封禁过期时间（ISO 8601格式），空字符串表示永久封禁 |

### 5.10 玩家踢出（`minecraft:players/kick`）

将指定玩家踢出服务器。

**请求参数：**

```json
{
  "jsonrpc": "2.0",
  "method": "minecraft:players/kick",
  "params": [{
    "player": [{"id": "uuid", "name": "playerName"}],
    "message": "踢出原因"
  }],
  "id": 1
}
```

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `player` | 数组 | 是 | 要踢出的玩家列表（PlayerDto 数组）|
| `message` | 字符串 | 否 | 踢出时显示给玩家的消息 |

**响应：** 返回实际被踢出的玩家列表。

### 5.11 服务端消息发送（`minecraft:server/system_message`）

向玩家发送服务端消息，可选择在聊天栏或动作栏显示。

**请求参数：**

```json
{
  "jsonrpc": "2.0",
  "method": "minecraft:server/system_message",
  "params": [{
    "message": {
      "literal": "要发送的文本",
      "translatable": "chat.type.text",
      "translatableParams": ["参数1", "参数2"]
    },
    "overlay": false,
    "receivingPlayers": [{"id": "uuid", "name": "playerName"}]
  }],
  "id": 1
}
```

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `message.literal` | 字符串 | 条件 | 要发送的纯文本（与 translatable 二选一）|
| `message.translatable` | 字符串 | 条件 | 本地化键名（如 `chat.type.text`）|
| `message.translatableParams` | 数组 | 否 | 本地化文本的参数替换列表 |
| `overlay` | 布尔 | 是 | 是否在动作栏显示（`true`）而非聊天栏（`false`）|
| `receivingPlayers` | 数组 | 否 | 指定接收消息的玩家列表，不指定则发送给所有玩家 |

**响应：** 布尔值，表示是否有玩家接收到了消息。

### 5.12 服务端设置管理（`minecraft:serversettings`）

服务端设置相关方法共有 40 个，用于查询和修改 `server.properties` 中的配置项。

| 配置项 | 查询方法 | 设置方法 |
|--------|----------|----------|
| 自动保存 | `serversettings/autosave` | `serversettings/autosave/set` |
| 世界难度 | `serversettings/difficulty` | `serversettings/difficulty/set` |
| 强制执行白名单 | `serversettings/enforce_allowlist` | `serversettings/enforce_allowlist/set` |
| 启用白名单 | `serversettings/use_allowlist` | `serversettings/use_allowlist/set` |
| 最大玩家数 | `serversettings/max_players` | `serversettings/max_players/set` |
| 无玩家暂停秒数 | `serversettings/pause_when_empty_seconds` | `serversettings/pause_when_empty_seconds/set` |
| 玩家空闲超时 | `serversettings/player_idle_timeout` | `serversettings/player_idle_timeout/set` |
| 允许飞行 | `serversettings/allow_flight` | `serversettings/allow_flight/set` |
| 服务器 MOTD | `serversettings/motd` | `serversettings/motd/set` |
| 出生点保护半径 | `serversettings/spawn_protection_radius` | `serversettings/spawn_protection_radius/set` |
| 强制游戏模式 | `serversettings/force_game_mode` | `serversettings/force_game_mode/set` |
| 默认游戏模式 | `serversettings/game_mode` | `serversettings/game_mode/set` |
| 渲染距离 | `serversettings/view_distance` | `serversettings/view_distance/set` |
| 模拟距离 | `serversettings/simulation_distance` | `serversettings/simulation_distance/set` |
| 接受转移连接 | `serversettings/accept_transfers` | `serversettings/accept_transfers/set` |
| 心跳间隔 | `serversettings/status_heartbeat_interval` | `serversettings/status_heartbeat_interval/set` |
| 管理员权限等级 | `serversettings/operator_user_permission_level` | `serversettings/operator_user_permission_level/set` |
| 隐藏在线玩家 | `serversettings/hide_online_players` | `serversettings/hide_online_players/set` |
| 状态回复 | `serversettings/status_replies` | `serversettings/status_replies/set` |
| 实体广播范围 | `serversettings/entity_broadcast_range` | `serversettings/entity_broadcast_range/set` |

所有查询方法返回对应配置项的当前值，设置方法接受新值作为参数并返回设置后的值。

---

## 6. 通知事件系统

### 6.1 通知机制设计

#### 6.1.1 服务端状态变更的实时推送

MSMP 的通知机制是其相对于传统管理协议的核心优势。与轮询模式相比，通知驱动架构具有显著的效率和实时性优势：

| 对比维度 | 轮询模式 | **通知模式** |
|---------|---------|-----------|
| **延迟** | 轮询间隔决定，通常秒级 | **亚秒级**，事件发生后立即推送 |
| **网络开销** | 与轮询频率成正比，大量无效请求 | **仅在有事件时产生流量** |
| **服务端负载** | 随客户端数量线性增长 | **与客户端数量无关**，事件驱动 |
| **实现复杂度** | 客户端简单，服务端无特殊要求 | 需要持久连接和状态管理 |
| **可靠性** | 可能错过短暂状态变化 | 依赖连接稳定性，可设计重放机制 |

WebSocket 的全双工特性为通知机制提供了理想的传输基础。服务端可以在任意时刻向客户端推送消息，无需等待请求 。

#### 6.1.2 通知与轮询的效率对比

以玩家在线状态监控为例：
- **RCON 轮询方案**：每秒执行 `list` 命令，解析文本输出，对比前后差异。10 个监控客户端 = 服务端每秒处理 10 条命令。
- **MSMP 通知方案**：玩家加入/离开时，服务端推送一次通知。10 个监控客户端 = 服务端推送 10 条消息，但仅在事件发生时。

对于 100 人在线、日均 500 次加入/离开的服务器，RCON 方案日处理 864,000 条命令，MSMP 方案仅处理 5,000 条通知——**效率提升 170 倍**。

### 6.2 标准通知事件类型

#### 6.2.1 服务端状态通知

| 通知类型 | 触发条件 | 参数 |
|---------|---------|------|
| `minecraft:notification/server/started` | 服务端加载完毕，开始运行游戏主循环前 | 无 |
| `minecraft:notification/server/stopping` | 服务端接收到停止指令，开始执行停止前 | 无 |
| `minecraft:notification/server/activity` | 客户端成功建立连接并开始握手流程（30秒内最多一次）| 无 |
| `minecraft:notification/server/saving` | 服务端保存所有区块前 | 无 |
| `minecraft:notification/server/saved` | 服务端保存所有区块后 | 无 |

#### 6.2.2 玩家生命周期通知

| 通知类型 | 触发条件 | 参数结构 |
|---------|---------|---------|
| `minecraft:notification/players/joined` | 玩家加入服务器 | 玩家对象（`id`, `name`）|
| `minecraft:notification/players/left` | 玩家离开服务器 | 玩家对象 |

```json
// 玩家加入通知示例
{
  "jsonrpc": "2.0",
  "method": "minecraft:notification/players/joined",
  "params": [{"id": "853c80ef-3c37-49fd-aa49-938b674adae6", "name": "jeb_"}]
}
```

#### 6.2.3 管理员列表变更通知

| 通知类型 | 触发条件 | 参数结构 |
|---------|---------|---------|
| `minecraft:notification/operators/added` | 管理员列表新增管理员 | OperatorDto |
| `minecraft:notification/operators/removed` | 管理员列表移除管理员 | OperatorDto |

#### 6.2.4 白名单变更通知

| 通知类型 | 触发条件 | 参数结构 |
|---------|---------|---------|
| `minecraft:notification/allowlist/added` | 白名单新增玩家 | 玩家对象 |
| `minecraft:notification/allowlist/removed` | 白名单移除玩家 | 玩家对象 |

#### 6.2.5 封禁列表变更通知

| 通知类型 | 触发条件 | 参数结构 |
|---------|---------|---------|
| `minecraft:notification/ip_bans/added` | 封禁 IP 列表新增 IP | IpBanDto |
| `minecraft:notification/ip_bans/removed` | 封禁 IP 列表移除 IP | IP 地址字符串 |
| `minecraft:notification/bans/added` | 封禁玩家列表新增玩家 | UserBanDto |
| `minecraft:notification/bans/removed` | 封禁玩家列表移除玩家 | 玩家对象 |

#### 6.2.6 游戏规则变更通知

| 通知类型 | 触发条件 | 参数结构 |
|---------|---------|---------|
| `minecraft:notification/game-rules/updated` | 游戏规则被修改 | TypedRule（包含 `key`, `type`, `value`）|

```json
// 游戏规则更新通知示例
{
  "jsonrpc": "2.0",
  "method": "minecraft:notification/game-rules/updated",
  "params": [{"key": "doDaylightCycle", "type": "boolean", "value": false}]
}
```

#### 6.2.7 服务端心跳通知

| 通知类型 | 触发条件 | 参数结构 |
|---------|---------|---------|
| `minecraft:notification/server/status` | 服务端发送心跳时（需配置 `status-heartbeat-interval`）| ServerState |

### 6.3 自定义通知扩展（未来预留）

MSMP 的命名空间预留机制为自定义扩展提供了规范路径。第三方服务器软件和模组可以使用**非保留命名空间**（即非 `minecraft` 和 `notification`）添加专属方法和通知。例如，Paper 服务器可以使用 `paper:` 命名空间，Fabric 模组可以使用模组 ID 作为命名空间前缀。

**Not Enough Management**（Fabric 模组）是社区扩展的实践案例，为原版 MSMP 添加了聊天消息通知等扩展功能 ：

| 扩展通知 | 来源 | 参数结构 |
|---------|------|---------|
| `nem:chat_message` | Not Enough Management | `id`, `name`, `message` |

扩展的最佳实践包括：使用反向域名风格的命名空间避免冲突；复用标准模式中的类型定义；在文档中明确说明扩展的依赖条件。

---

## 7. API 模式与代码生成

### 7.1 静态模式文件获取

#### 7.1.1 数据生成器运行：`--reports` 参数

数据生成器（Data Generator）是 Minecraft 内置的工具，用于导出游戏数据的各种结构化表示。运行数据生成器获取 API 模式：

```bash
java -jar server.jar --reports
```

#### 7.1.2 `json-rpc-api-schema.json` 输出位置

输出目录中将包含 `reports/json-rpc-api-schema.json` 文件，其内容与运行时 `rpc.discover` 响应一致 。

### 7.2 模式文件结构解析

| 键 | 内容 |
|---|------|
| `methods` | 支持的方法列表，每个包含名称、参数模式、返回模式 |
| `notifications` | 支持的通知列表，每个包含名称、参数模式 |
| `components/schemas` | 可复用的类型定义，JSON Schema 格式 |

### 7.3 客户端代码生成实践

基于 JSON Schema 的代码生成可以显著提高开发效率和类型安全性。Rust 社区的 `mc-server-management` 实现展示了这一实践，利用 `schemars` 和 `serde` crate 从模式生成强类型客户端 。

生成流程：
1. 解析 `json-rpc-api-schema.json`
2. 为每个方法生成请求/响应结构体
3. 为每个通知生成事件处理类型
4. 生成客户端 trait 或结构体，封装 WebSocket 连接管理

---

## 8. 插件开发最佳实践

### 8.1 连接管理

#### 8.1.1 连接池与长连接维护

MSMP 设计为**长连接协议**，客户端应保持单一连接而非频繁创建销毁。连接建立后，通过同一连接发送所有请求并接收通知。

#### 8.1.2 断线重连与指数退避策略

网络不稳定场景需要健壮的断线重连机制。推荐采用**指数退避策略**：

| 重试次数 | 等待间隔 | 累计等待 |
|---------|---------|---------|
| 1 | 1s | 1s |
| 2 | 2s | 3s |
| 3 | 4s | 7s |
| 4 | 8s | 15s |
| 5+ | 60s（上限）| - |

首次断线立即重试，后续间隔递增，避免对服务端造成重连风暴。

#### 8.1.3 心跳检测与连接健康监控

虽然 WebSocket 协议内置 Ping/Pong 帧，但应用层可以实现更语义化的心跳。定期发送 `minecraft:server/status` 请求，既可以检测连接健康，又可以同步服务端状态。

### 8.2 认证安全

#### 8.2.1 密钥安全存储：环境变量 vs 配置文件

| 存储方式 | 安全性 | 便利性 | 适用场景 |
|---------|--------|--------|---------|
| **环境变量** | ⭐⭐⭐ 高 | ⭐⭐ 中 | **生产环境推荐** |
| Docker Secrets/K8s Secrets | ⭐⭐⭐ 高 | ⭐⭐ 中 | 容器化部署 |
| 配置文件（加密）| ⭐⭐ 中 | ⭐⭐⭐ 高 | 小型部署 |
| 配置文件（明文）| ⭐ 低 | ⭐⭐⭐ 高 | **仅限开发** |

#### 8.2.2 密钥轮换与最小权限原则

- **密钥轮换**：MSMP 当前版本不支持动态密钥轮换，需要重启服务端生效。规划中的改进包括热重载配置和密钥版本控制。
- **最小权限**：为不同用途的客户端分配不同密钥（未来版本支持），或通过网络隔离限制访问范围。

#### 8.2.3 生产环境强制 TLS 配置

| 检查项 | 要求 |
|--------|------|
| TLS 启用 | `management-server-tls-enabled=true` |
| 有效证书 | 公共 CA 签发或受信任的自签名 |
| 证书有效期 | 监控过期时间，自动续期 |
| 协议版本 | TLS 1.2+，禁用 SSLv3/TLS 1.0/1.1 |

### 8.3 错误处理与容错

#### 8.3.1 JSON-RPC 错误码分类处理

| 错误码范围 | 类型 | 处理策略 |
|-----------|------|---------|
| `-32700` ~ `-32600` | 协议错误 | 记录日志，排查请求构造 |
| `-32601` | 方法不存在 | 检查版本兼容性，优雅降级 |
| `-32602` | 参数错误 | 验证输入，提示用户 |
| `-32603` | 内部错误 | 可能临时故障，可重试 |
| `-32000` ~ `-32099` | 服务端错误 | 监控告警，人工介入 |

#### 8.3.2 网络超时与重试策略

为每个请求设置合理的**超时时间（建议 30s）**，避免无限等待。超时后应视为连接异常，触发重连。

#### 8.3.3 服务端重启后的状态同步

服务端重启后，客户端应：
1. 重新执行 `rpc.discover` 获取当前能力
2. 刷新所有缓存状态（玩家列表、白名单、游戏规则等）
3. 重新注册通知处理器（若服务端状态丢失）

### 8.4 性能优化

#### 8.4.1 批量操作优先于单条调用

| 操作 | 单条调用（10次）| 批量调用（1次）|
|------|--------------|-------------|
| 白名单添加 | 10 次往返 | 1 次往返 |
| 管理员设置 | 10 次往返 | 1 次往返 |

**批量操作减少 90% 网络延迟**。

#### 8.4.2 通知订阅替代主动轮询

| 数据类型 | 传统方案 | MSMP 优化方案 |
|---------|---------|-------------|
| 玩家在线状态 | 每秒 `list` 命令 | 订阅 `minecraft:notification/players/joined` + `minecraft:notification/players/left` |
| 游戏规则 | 定期全量查询 | 订阅 `minecraft:notification/game-rules/updated` |
| 白名单 | 定期全量查询 | 本地缓存 + 增量更新 |

#### 8.4.3 本地缓存与增量更新策略

- **缓存策略**：合理设置 TTL，游戏规则等不频繁变化数据可长期缓存
- **增量更新**：收到通知后，更新本地缓存而非重新查询
- **失效处理**：连接断开后，标记缓存失效，重连后全量刷新

### 8.5 调试与监控

#### 8.5.1 请求/响应日志记录

| 环境 | 记录内容 |
|------|---------|
| 开发 | 完整请求/响应 JSON |
| 生产 | 方法名、响应时间、错误码、数据大小 |

#### 8.5.2 性能指标采集

| 指标 | 说明 | 告警阈值 |
|-----|------|---------|
| 连接建立时间 | WebSocket 握手完成时间 | > 5s |
| 请求延迟（P50/P99）| 请求到响应的时间 | P99 > 1s |
| 通知延迟 | 事件发生到推送到达的时间 | > 100ms |
| 错误率 | 失败请求占总请求比例 | > 1% |
| 重连频率 | 单位时间重连次数 | > 10/分钟 |

---

## 9. 安全注意事项与风险防控

### 9.1 网络暴露风险

#### 9.1.1 公网暴露的 TLS 强制要求

MSMP 提供的服务器控制能力使其成为**高价值攻击目标**。公网暴露必须满足：

| 层级 | 措施 |
|-----|------|
| 传输层 | **TLS 强制启用**，使用有效证书 |
| 认证层 | 强认证密钥（40位随机字符）|
| 应用层 | 来源验证（浏览器场景）|
| 网络层 | 防火墙/安全组访问控制 |

#### 9.1.2 反向代理部署模式

推荐在生产环境使用反向代理（如 Nginx、Traefik）终止 TLS，提供额外的：
- **访问日志**：集中审计
- **速率限制**：防止暴力破解
- **WAF 保护**：过滤恶意请求

### 9.2 认证密钥保护

#### 9.2.1 密钥泄露的潜在危害：完整服务器控制

密钥泄露意味着攻击者获得：
- 修改白名单和管理员列表
- 执行服务器停止命令
- 更改游戏规则和服务器设置
- **潜在的权限提升路径**

#### 9.2.2 密钥生成与分发流程

| 阶段 | 最佳实践 |
|-----|---------|
| 生成 | 加密安全随机数生成器（`crypto.randomBytes` 等）|
| 分发 | 安全通道（SSH、密钥管理服务）|
| 存储 | 环境变量或专用密钥管理系统 |
| 轮换 | 定期更换，制定应急响应流程 |

### 9.3 来源验证绕过风险

`allowed-origins` 配置不当可能导致**跨站 WebSocket 劫持（CSWSH）**。严格遵循：
- **明确列出**所有授权来源，不使用通配符
- **定期审计**来源列表，移除不再使用的条目
- **监控 401 响应日志**，检测潜在探测行为

### 9.4 权限最小化原则

| 原则 | 实践 |
|-----|------|
| 方法级最小权限 | 仅请求必要的 API 方法 |
| 读写分离 | 只读操作与写操作分离到不同插件 |
| 敏感操作确认 | `server/stop` 等操作需要额外确认或审批 |

---

## 10. 生态工具与社区资源

### 10.1 官方与半官方库

| 库名称 | 语言 | 维护者 | 特点 | 状态 |
|--------|------|--------|------|------|
| `mc-server-management` | **Node.js** | Aternos | 高级抽象，自动缓存，通知同步 | 活跃维护  |
| `mc-server-management` | **Rust** | 社区 | 类型安全，异步运行时 | 活跃维护  |

### 10.2 扩展模组

**Not Enough Management**（Fabric）
- 功能：为原版 MSMP 添加聊天消息通知等扩展
- 技术：自定义命名空间 `nem:chat_message`
- 参考价值：展示 MSMP 扩展机制的实践 

### 10.3 第三方管理面板集成

| 平台 | 集成状态 | 说明 |
|-----|---------|------|
| **Aternos** | ✅ 已集成 | 利用 MSMP 原生支持提供可靠管理体验  |
| 其他托管平台 | 评估中 | 预计 2025-2026 年逐步支持 |

### 10.4 相关协议对比

#### 10.4.1 MSMP vs RCON：功能、安全、实时性

| 维度 | MSMP | RCON |
|-----|------|------|
| **功能覆盖** | 完整 CRUD + 实时通知 | 仅命令执行 |
| **安全性** | TLS + 40位密钥 + Origin 验证 | 明文密码 |
| **实时性** | 事件驱动，<10ms 延迟 | 轮询依赖，秒级延迟 |
| **迁移建议** | 新项目的首选方案 | 逐步迁移，利用通知替代轮询 |

#### 10.4.2 MSMP vs Query：适用场景差异

| 场景 | 推荐协议 |
|-----|---------|
| 服务器列表网站展示 | **Query**（无认证，UDP 穿透性好）|
| 实时管理面板 | **MSMP** |
| 第三方工具兼容 | Query 与 MSMP 并存 |

#### 10.4.3 MSMP vs 第三方 REST API 插件

| 维度 | MSMP | 第三方 REST API |
|-----|------|---------------|
| 官方支持 | ✅ 原生集成 | ❌ 社区维护 |
| 版本同步 | ✅ 保证 | ⚠️ 不确定 |
| 功能完整性 | ✅ 完整 | 依赖插件 |
| 安全性 | ✅ 标准化 | 依赖实现 |
| **建议** | **新项目的首选** | 评估迁移成本 |

---

## 11. 历史版本更新记录

| 版本 | 快照 | 发布日期 | 更新内容 |
|------|------|----------|----------|
| **1.21.9** | 25w35a | 2025-08-26 | **首次加入**服务端管理协议（MSMP）|
| - | 25w37a | - | 加入强制认证机制；支持 TLS 加密；支持批量请求格式 |
| - | pre1 | - | 通知请求命名空间从 `notification:` 改为 `minecraft:notification/` |
| - | pre2 | - | 支持按名称指定参数 |
| **1.21.11** | 25w41a | - | 协议版本号更新为 `1.1.0`；添加 `server/activity` 通知 |
| - | 25w42a | - | 允许使用浏览器 WebSocket API 连入 |
| - | 25w44a | - | 协议版本号更新为 `2.0.0`；修改游戏规则管理相关的请求和响应格式 |

---

## 12. 常见问题与故障排查

### 12.1 连接问题

#### 12.1.1 401 Unauthorized：认证失败排查清单

| 检查项 | 验证方法 |
|--------|---------|
| 密钥正确复制 | 无多余空格或换行 |
| 密钥长度 40 位 | 字符计数验证 |
| 认证头格式 | `Bearer ` 前缀或 `minecraft-v1,` 前缀 |
| 浏览器来源白名单 | `allowed-origins` 包含页面 Origin |
| 服务端日志 | 查看具体拒绝原因 |

#### 12.1.2 连接被拒绝：端口、防火墙、绑定地址

| 检查层级 | 命令/方法 |
|---------|----------|
| 服务端绑定状态 | `ss -tlnp | grep :<port>` |
| 操作系统防火墙 | `firewall-cmd --list-ports`、`ufw status` |
| 云安全组 | 控制台检查入站规则 |
| 绑定地址匹配 | 确认连接地址与绑定地址一致 |

### 12.2 调用问题

#### 12.2.1 Method not found：命名空间拼写、服务端版本

排查清单：
- 检查命名空间拼写（`minecraft` 而非 `minecrafts`）
- 验证方法路径（`allowlist/add` 而非 `allowlist.add`）
- 确认服务端版本支持该方法
- **运行 `rpc.discover` 获取实际支持的方法列表**

#### 12.2.2 参数格式错误：数组包裹、对象字段

| 常见错误 | 正确格式 |
|---------|---------|
| `params: {"name": "player"}` | `params: [[{"name": "player"}]]` |
| 批量操作单层数组 | 双层数组 `[[{...}, {...}]]` |

### 12.3 性能问题

#### 12.3.1 高频率调用的限流与缓存

| 优化策略 | 实施方法 |
|---------|---------|
| 通知替代轮询 | 订阅事件而非定时查询 |
| 批量操作合并 | 累积单条请求，定时批量发送 |
| 本地缓存 | 设置合理 TTL，增量更新 |
| 请求限流 | 客户端自我约束，避免服务端过载 |

---

## 13. 附录

### 13.1 完整配置项速查表

| 配置项 | 默认值 | 类型 | 说明 |
|--------|--------|------|------|
| `management-server-enabled` | `false` | 布尔 | **启用开关** |
| `management-server-host` | `localhost` | 字符串 | 绑定地址 |
| `management-server-port` | `0` | 整数 | 监听端口，0=随机 |
| `management-server-secret` | 自动生成 | 字符串(40) | **认证密钥** |
| `management-server-tls-enabled` | `true` | 布尔 | **TLS开关** |
| `management-server-tls-keystore` | 无 | 字符串 | PKCS12密钥库路径 |
| `management-server-tls-keystore-password` | 无 | 字符串 | 密钥库密码（多来源）|
| `management-server-allowed-origins` | 无 | 逗号分隔字符串 | **浏览器来源白名单** |
| `status-heartbeat-interval` | `0` | 整数 | 心跳通知间隔（秒），0=禁用 |

### 13.2 HTTP/1.1 与 WebSocket 协议细节

MSMP 要求 **HTTP/1.1 作为 WebSocket 握手的传输层** 。WebSocket 握手请求示例：

**`Authorization` 认证方式：**
```http
GET / HTTP/1.1
Host: localhost:8080
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Authorization: Bearer n2pQcIG1OQ92jot2xG1M0aw0ZWnrh4F3Z3jw8qRP
```

**`Sec-WebSocket-Protocol` 认证方式（浏览器）：**
```http
GET / HTTP/1.1
Host: localhost:8080
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: minecraft-v1,n2pQcIG1OQ92jot2xG1M0aw0ZWnrh4F3Z3jw8qRP
Origin: http://localhost:3000
```

### 13.3 JSON-RPC 2.0 规范参考链接

| 资源 | 链接 |
|-----|------|
| JSON-RPC 2.0 规范 | https://www.jsonrpc.org/specification |
| Minecraft Wiki - 服务端管理协议（中文）| https://zh.minecraft.wiki/w/服务端管理协议 |
| Minecraft Wiki - MSMP（英文）| https://minecraft.wiki/w/Minecraft_Server_Management_Protocol |