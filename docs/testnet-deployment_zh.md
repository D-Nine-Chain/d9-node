# D9 测试网部署指南

本指南提供了部署具有多个验证节点的 D9 测试网络的分步说明。

## 目录

- [前提条件](#前提条件)
- [网络架构](#网络架构)
- [网络端口和 RPC/WebSocket 配置](#网络端口和-rpcwebsocket-配置)
- [部署步骤](#部署步骤)
  - [1. 准备链规范](#1-准备链规范)
  - [2. 部署引导节点](#2-部署引导节点)
  - [3. 部署验证节点](#3-部署验证节点)
  - [4. 生成并插入会话密钥](#4-生成并插入会话密钥)
  - [5. 配置验证节点](#5-配置验证节点)
  - [6. 启动网络](#6-启动网络)
- [验证](#验证)
- [故障排除](#故障排除)

## 前提条件

### 系统要求（每个节点）

- **操作系统**：Ubuntu 22.04 LTS 或 Debian 12
- **架构**：x86_64 或 ARM64
- **内存**：最低 8GB（推荐 16GB）
- **存储**：最低 60GB 可用空间（推荐 SSD）
- **网络**：稳定的互联网连接，开放端口
- **所需端口**：
  - 40100（P2P 通信）
  - 40200（RPC - 可选，用于监控）

### 软件要求

- D9 节点二进制文件（从源代码构建或下载）
- jq（用于 JSON 处理）
- SSH 和 Linux 命令行的基础知识

## 网络架构

### 理解节点类型和要求

D9 网络由不同类型的节点组成，每种节点都有特定的用途和要求：

#### 1. **引导节点**（Bootstrap Node）
- **用途**：新节点发现网络的入口点
- **所需密钥**：❌ **无** - 引导节点不需要验证者密钥
- **功能**：帮助节点相互发现（对等发现）
- **可以是**：专用轻量级节点或您的验证节点之一可以充当引导节点
- **建议**：部署专用引导节点以获得更清晰的架构

#### 2. **验证节点**
- **用途**：生成和最终确定区块，维护共识
- **所需密钥**：✅ **是** - 验证节点必须拥有所有会话密钥：
  - **Aura**（Sr25519）- 区块生成权限
  - **Grandpa**（Ed25519）- 区块最终确定
  - **ImOnline**（Sr25519）- 心跳/活跃性证明
- **最低要求**：**3 个验证节点**才能使网络正常运行
- **建议**：从 3-4 个验证节点开始测试

#### 3. **完整节点**（非验证）
- **用途**：同步和存储区块链数据，可以提供 RPC 请求
- **所需密钥**：❌ **无** - 完整节点不需要验证者密钥
- **功能**：提供冗余，充当 RPC 端点
- **使用场景**：公共 RPC/WSS 访问点

#### 4. **RPC 节点**（具有公共访问的完整节点）
- **用途**：为用户和应用程序提供公共 RPC/WebSocket 访问
- **所需密钥**：❌ **无** - RPC 节点不需要验证者密钥
- **功能**：处理用户交易和查询
- **安全性**：应运行 `--rpc-methods Safe` 并使用 Nginx 反向代理

### 最小可行网络配置

#### 选项 1：绝对最小（3 个节点）
```
节点 1：验证节点 + 引导节点（需要密钥：Aura、Grandpa、ImOnline）
节点 2：验证节点（需要密钥：Aura、Grandpa、ImOnline）
节点 3：验证节点（需要密钥：Aura、Grandpa、ImOnline）
```
**总计**：3 个验证节点（一个兼作引导节点）
- ✅ 网络将正常运行
- ⚠️ 没有专用 RPC 端点
- ⚠️ 验证节点提供 RPC 不推荐用于生产环境

#### 选项 2：推荐最小值（4 个节点）
```
节点 1：引导节点（不需要密钥）
节点 2：验证节点（需要密钥：Aura、Grandpa、ImOnline）
节点 3：验证节点（需要密钥：Aura、Grandpa、ImOnline）
节点 4：验证节点（需要密钥：Aura、Grandpa、ImOnline）
```
**总计**：1 个引导节点 + 3 个验证节点
- ✅ 更清晰的关注点分离
- ✅ 网络将正常运行
- ⚠️ 仍然没有专用 RPC 端点

#### 选项 3：功能性部署（5 个节点）⭐ 推荐
```
节点 1：引导节点（不需要密钥）
节点 2：验证节点（需要密钥：Aura、Grandpa、ImOnline）
节点 3：验证节点（需要密钥：Aura、Grandpa、ImOnline）
节点 4：验证节点（需要密钥：Aura、Grandpa、ImOnline）
节点 5：RPC/WSS 节点（不需要密钥）
```
**总计**：1 个引导节点 + 3 个验证节点 + 1 个 RPC 节点
- ✅ 合适的网络架构
- ✅ 专用公共访问点
- ✅ 验证节点受保护
- ✅ 用户可以通过 RPC 节点进行交互
- ✅ 生产就绪设置

### 密钥要求摘要

| 节点类型 | 需要 Aura 密钥? | 需要 Grandpa 密钥? | 需要 ImOnline 密钥? | 总密钥数 |
|-----------|----------------|-------------------|-------------------|------------|
| **引导节点** | ❌ 否 | ❌ 否 | ❌ 否 | 0 |
| **验证节点** | ✅ 是（Sr25519） | ✅ 是（Ed25519） | ✅ 是（Sr25519） | 3 |
| **完整节点** | ❌ 否 | ❌ 否 | ❌ 否 | 0 |
| **RPC 节点** | ❌ 否 | ❌ 否 | ❌ 否 | 0 |

**重要注意事项：**
- 只有验证节点需要密钥 - 它们必须拥有全部三个会话密钥
- 每个验证节点需要一组唯一的密钥（不同的助记词）
- 密钥由助记词生成并插入到节点的密钥库中
- 没有正确的密钥，验证节点无法参与共识
- 引导节点和 RPC 节点永远不需要密钥

### 为什么最少需要 3 个验证节点？

D9 使用的共识机制要求：
- **2/3 + 1** 验证节点同意才能最终确定（Grandpa）
- 有 3 个验证节点：需要 3 × 2/3 = 2 个验证节点 + 1 = **最少 3 个验证节点**
- 有 2 个验证节点：无法容忍任何故障（2 × 2/3 = 1.33，向上取整 = 2，但需要 +1）
- **少于 3 个验证节点 = 网络无法最终确定区块**

### 典型生产网络（推荐）

对于生产测试网，请考虑以下架构：

```
┌─────────────────────────────────────────────────────────────┐
│                    互联网 / 用户                            │
│                （钱包、dApp、区块浏览器）                   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ WSS (443) 通过 Nginx
                         │
                ┌────────▼────────┐
                │   RPC 节点      │  （不需要密钥）
                │  （公共 WSS）   │  --rpc-methods Safe
                └────────┬────────┘  --rpc-external
                         │           --ws-external
                         │
                         │ P2P (40100)
                         │
        ┌────────────────┴────────────────┐
        │                                 │
┌───────▼────────┐              ┌────────▼────────┐
│   引导节点     │◄────────────►│   验证节点 1    │
│  （不需要密钥）│              │ (Aura, Grandpa, │
│                │              │   ImOnline)     │
└───────┬────────┘              └────────┬────────┘
        │                                │
        │         P2P 网络               │
        │        (40100)                 │
        │                                │
┌───────▼────────┐              ┌────────▼────────┐
│  验证节点 2    │◄────────────►│   验证节点 3    │
│ (Aura, Grandpa,│              │ (Aura, Grandpa, │
│   ImOnline)    │              │   ImOnline)     │
└────────────────┘              └─────────────────┘
```

**此设置提供：**
- ✅ 网络可以达成共识（3 个验证节点）
- ✅ 轻松的对等发现（专用引导节点）
- ✅ 用户的公共访问（RPC 节点）
- ✅ 验证节点受保护（没有公共 RPC）
- ✅ 可扩展（可以添加更多验证节点和 RPC 节点）

## 网络端口和 RPC/WebSocket 配置

### 端口类型说明

D9 节点使用三种主要类型的端口进行不同的通信目的：

#### 1. **P2P 端口（点对点）** - 默认：`40100`
- **用途**：区块链共识的节点间通信
- **协议**：TCP
- **使用**：验证节点和完整节点相互发现、同步区块、交换交易
- **必须开放给**：互联网（公共）用于节点发现
- **安全性**：无敏感数据，加密的对等通信
- **标志**：`--port 40100`

#### 2. **RPC 端口（HTTP RPC）** - 默认：`9933`
- **用途**：用于区块链查询和交易的 HTTP JSON-RPC API
- **协议**：HTTP over TCP
- **使用**：应用程序、脚本、监控工具发出一次性请求
- **应该开放给**：仅本地主机（除非您需要外部访问）
- **安全性**：可能暴露敏感操作 - 对公共暴露使用 `--rpc-methods Safe`
- **标志**：`--rpc-port 9933`

#### 3. **WebSocket 端口（WS RPC）** - 默认：`9944`
- **用途**：用于实时区块链订阅的 WebSocket JSON-RPC API
- **协议**：WebSocket（WS）或安全 WebSocket（WSS）over TCP
- **使用**：需要实时更新的钱包、dApp、区块浏览器（区块通知、事件订阅）
- **应该开放给**：取决于使用场景（开发用本地主机，生产用带反向代理的公共）
- **安全性**：与 RPC 相同 - 限制公共访问的方法
- **标志**：`--ws-port 9944`

### 端口配置矩阵

| 节点类型 | P2P (40100) | RPC (9933) | WS (9944) | 外部访问 |
|-----------|-------------|------------|-----------|-----------------|
| **验证节点** | 开放（公共） | 关闭 | 关闭 | 无（安全） |
| **完整节点（私有）** | 开放（公共） | 仅本地主机 | 仅本地主机 | 无 |
| **RPC 节点（公共）** | 开放（公共） | 开放（受限） | 开放（受限） | 是（带 CORS） |
| **开发** | 本地主机 | 本地主机 | 本地主机 | 无 |

### 配置标志参考

#### 基本端口配置
```bash
--port 40100          # P2P 端口用于节点间通信
--rpc-port 9933       # HTTP RPC 端口
--ws-port 9944        # WebSocket 端口
```

#### 对外暴露端口
```bash
--rpc-external        # 允许外部 RPC 连接（所有接口：0.0.0.0）
--ws-external         # 允许外部 WebSocket 连接（所有接口：0.0.0.0）
--unsafe-rpc-external # 与 --rpc-external 相同但抑制安全警告
--unsafe-ws-external  # 与 --ws-external 相同但抑制安全警告
```

**警告**：使用 `--rpc-external` 或 `--ws-external` 会将您的节点暴露给互联网。始终与 CORS 和方法限制结合使用！

#### CORS 配置（跨源资源共享）
```bash
--rpc-cors all                    # 允许所有来源（危险 - 仅用于开发！）
--rpc-cors "https://app.d9.network"  # 允许特定域
--rpc-cors "https://app.d9.network,https://wallet.d9.network"  # 多个域
--rpc-cors "null"                 # 允许 file:// 协议（本地 HTML 文件）
```

**默认值**：如果未指定，仅允许 `localhost` 和 `https://polkadot.js.org`。

#### 方法限制
```bash
--rpc-methods Safe    # 仅允许安全的只读方法（推荐用于公共节点）
--rpc-methods Unsafe  # 允许所有方法，包括状态更改操作（默认）
```

**安全方法包括**：读取链状态、查询区块、订阅事件
**不安全方法包括**：提交密钥、插入数据、开发者 API

#### 连接限制
```bash
--ws-max-connections 1000  # 最大并发 WebSocket 连接（默认：100）
--in-peers 25              # 最大传入 P2P 连接（默认：25）
--out-peers 25             # 最大传出 P2P 连接（默认：25）
```

### 配置示例

#### 示例 1：验证节点（安全 - 无外部访问）
```bash
./target/release/d9-node \
  --base-path /home/ubuntu/node-data \
  --chain /usr/local/bin/testnet-spec.json \
  --name "D9-Validator-1" \
  --validator \
  --port 40100
  # 未暴露 RPC/WS - 仅可通过本地主机访问
```

**使用场景**：生产验证节点 - 最大安全性，无外部 RPC 访问

#### 示例 2：具有本地 RPC 访问的完整节点
```bash
./target/release/d9-node \
  --base-path /home/ubuntu/node-data \
  --chain /usr/local/bin/testnet-spec.json \
  --name "D9-FullNode" \
  --port 40100 \
  --rpc-port 9933 \
  --ws-port 9944
  # RPC/WS 仅在本地主机（127.0.0.1）上可用
```

**使用场景**：运行需要查询区块链的本地应用程序

#### 示例 3：公共 RPC 节点（开发测试网）
```bash
./target/release/d9-node \
  --base-path /home/ubuntu/node-data \
  --chain /usr/local/bin/testnet-spec.json \
  --name "D9-RPC-Public" \
  --port 40100 \
  --rpc-port 9933 \
  --ws-port 9944 \
  --rpc-external \
  --ws-external \
  --rpc-cors all \
  --ws-max-connections 1000
```

**使用场景**：为开发者提供的测试网公共 RPC（不推荐用于主网）

#### 示例 4：公共 RPC 节点（生产 - 受限）
```bash
./target/release/d9-node \
  --base-path /home/ubuntu/node-data \
  --chain /usr/local/bin/mainnet-spec.json \
  --name "D9-RPC-Prod" \
  --port 40100 \
  --rpc-port 9933 \
  --ws-port 9944 \
  --rpc-external \
  --ws-external \
  --rpc-methods Safe \
  --rpc-cors "https://app.d9.network,https://wallet.d9.network" \
  --ws-max-connections 500
```

**使用场景**：为您的 dApp/钱包提供受限访问的生产 RPC 节点

### 为 WSS（安全 WebSocket）设置反向代理（Nginx）

对于生产环境，您应该通过反向代理使用带 SSL/TLS 加密的 **WSS（安全 WebSocket）**。

#### 步骤 1：安装 Nginx
```bash
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx
```

#### 步骤 2：创建 Nginx 配置
创建 `/etc/nginx/sites-available/d9-rpc`：

```nginx
# HTTP 到 HTTPS 重定向
server {
    listen 80;
    server_name rpc.yourdomain.com;

    location / {
        return 301 https://$server_name$request_uri;
    }
}

# 带 WebSocket 代理的 HTTPS 服务器
server {
    listen 443 ssl http2;
    server_name rpc.yourdomain.com;

    # SSL 证书（将由 certbot 添加）
    ssl_certificate /etc/letsencrypt/live/rpc.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/rpc.yourdomain.com/privkey.pem;

    # SSL 配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # WebSocket 代理到 D9 节点
    location / {
        proxy_pass http://127.0.0.1:9944;
        proxy_http_version 1.1;

        # WebSocket 升级头
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # 标准代理头
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 长连接超时
        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }

    # 可选：HTTP RPC 端点
    location /rpc {
        proxy_pass http://127.0.0.1:9933;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

#### 步骤 3：启用站点并获取 SSL 证书
```bash
# 启用站点
sudo ln -s /etc/nginx/sites-available/d9-rpc /etc/nginx/sites-enabled/

# 测试配置
sudo nginx -t

# 获取 SSL 证书
sudo certbot --nginx -d rpc.yourdomain.com

# 重新加载 Nginx
sudo systemctl reload nginx
```

#### 步骤 4：更新防火墙
```bash
sudo ufw allow 'Nginx Full'
sudo ufw allow 40100/tcp  # P2P 端口
```

#### 步骤 5：配置 D9 节点（不需要外部标志！）
```bash
./target/release/d9-node \
  --base-path /home/ubuntu/node-data \
  --chain /usr/local/bin/mainnet-spec.json \
  --name "D9-RPC-Prod" \
  --port 40100 \
  --rpc-port 9933 \
  --ws-port 9944 \
  --rpc-methods Safe
  # 不需要 --rpc-external 或 --ws-external！
  # Nginx 处理外部访问
```

**连接 URL**：`wss://rpc.yourdomain.com`

### 连接到您的节点

#### 从 Polkadot.js Apps 连接
1. 打开 [https://polkadot.js.org/apps](https://polkadot.js.org/apps)
2. 点击左上角的网络下拉菜单
3. 选择"Development"→"Custom endpoint"
4. 输入您的端点：
   - 本地：`ws://127.0.0.1:9944`
   - 远程（使用 Nginx）：`wss://rpc.yourdomain.com`
   - 远程（直接，不安全）：`ws://YOUR_IP:9944`

#### 从 JavaScript/TypeScript 应用程序连接
```javascript
import { ApiPromise, WsProvider } from '@polkadot/api';

// 连接到本地节点
const wsProvider = new WsProvider('ws://127.0.0.1:9944');

// 连接到带 WSS 的远程节点（生产）
// const wsProvider = new WsProvider('wss://rpc.yourdomain.com');

const api = await ApiPromise.create({ provider: wsProvider });

// 订阅新区块
await api.rpc.chain.subscribeNewHeads((header) => {
  console.log(`Chain is at block: #${header.number}`);
});
```

#### 使用 Curl（HTTP RPC）
```bash
# 本地节点
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "system_health"}' \
  http://localhost:9933

# 远程节点
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "system_health"}' \
  https://rpc.yourdomain.com/rpc
```

## 理解区块链交互和交易提交

### 为什么您需要 RPC/WebSocket 访问

理解区块链交互的架构对于在 D9 上构建应用程序、钱包或服务的任何人都至关重要。

#### 两个通信层

**P2P 端口（40100）- 仅用于验证节点通信**
- 专门用于验证节点之间的通信
- 处理区块传播、共识消息和对等发现
- **不可用于用户应用程序**
- **不能用于提交交易或查询区块链状态**
- 只有验证节点和完整节点通过此端口通信

**RPC/WebSocket 端口（9933/9944）- 用户应用程序层**
- **所有用户与区块链的交互都需要**
- 处理交易提交、状态查询和事件订阅
- 这是钱包、dApp 和服务与 D9 交互的方式
- 读取（查询）和写入（交易）操作都使用这些端口

#### 为什么存在这种分离

```
┌─────────────────────────────────────────────────────────────┐
│                    用户应用程序                             │
│        （钱包、dApp、区块浏览器、您的应用程序）            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ RPC/WebSocket (9933/9944)
                         │ • 提交交易
                         │ • 查询区块链状态
                         │ • 订阅事件
                         │
┌────────────────────────▼────────────────────────────────────┐
│                    D9 区块链节点                            │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              RPC/WebSocket 接口                      │  │
│  │  • 接受用户交易                                     │  │
│  │  • 处理查询                                         │  │
│  │  • 管理订阅                                         │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │                Runtime（区块链逻辑）                 │  │
│  │  • D9 余额 Pallet                                   │  │
│  │  • D9 推荐系统                                       │  │
│  │  • 节点投票 Pallet                                  │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              P2P 网络接口                            │  │
│  │  • 区块传播                                         │  │
│  │  • 共识消息                                         │  │
│  │  • 对等发现                                         │  │
│  └─────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ P2P (40100)
                         │ 仅限验证节点
                         │
┌────────────────────────▼────────────────────────────────────┐
│              其他 D9 验证节点                               │
└─────────────────────────────────────────────────────────────┘
```

**关键要点**：如果您想与 D9 区块链交互（检查余额、发送代币、为验证节点投票），您**必须**通过 RPC/WebSocket 连接。P2P 端口不可用于此目的。

### 什么是连接器/客户端库？

要与 D9 交互，您需要一个**客户端库**（也称为连接器），它可以：

1. **管理连接**到您的节点（通过 WebSocket 或 HTTP）
2. **处理加密**以签署交易
3. **编码/解码数据**为区块链的正确格式
4. **提供开发者友好的 API**而不是原始 JSON-RPC 调用

#### 流行的基于 Substrate 链的客户端库

**对于 JavaScript/TypeScript（最常见）：**
- `@polkadot/api` - 官方 Polkadot.js API 库
- 功能齐全、文档完善、积极维护
- 适用于 Node.js 和浏览器
- **这是您将用于 D9 的库**

**对于其他语言：**
- Python：`substrate-interface`
- Rust：`subxt`
- Go：`gsrpc`
- Java：`polkaj`

### 读取与写入操作

理解读取数据和写入数据之间的区别至关重要：

#### 读取操作（查询）

**它们是什么：**
- 检索区块链状态（余额、存储等）
- 订阅事件（新区块、交易）
- 获取链元数据和信息

**要求：**
- ✅ RPC/WebSocket 连接
- ❌ 不需要账户/私钥
- ❌ 无交易费用
- ❌ 不需要签名

**示例：**
- 检查账户余额
- 获取当前区块号
- 查询验证节点投票
- 订阅新区块
- 读取推荐关系

#### 写入操作（交易/外部调用）

**它们是什么：**
- 任何改变区块链状态的操作
- 在 Substrate 术语中也称为"extrinsics"
- 必须由验证节点包含在区块中

**要求：**
- ✅ RPC/WebSocket 连接
- ✅ 具有私钥的账户（用于签名）
- ✅ 足够的余额用于交易费用
- ✅ 有效签名

**示例：**
- 转账代币
- 为验证节点投票
- 注册为验证节点候选人
- 更新验证节点佣金

**交易生命周期：**
```
1. 创建交易   → 您的应用程序
2. 签署交易   → 使用您的私钥
3. 提交到节点 → 通过 RPC/WebSocket
4. 进入交易池 → 节点的交易池
5. 验证节点选择交易 → 下一个区块作者
6. 包含在区块中 → 区块被创建
7. 区块被最终确定 → 交易是永久的
8. 发出事件 → 您可以订阅此事件
```

### D9 特定交易示例

D9 具有独特功能的自定义 pallet。以下是您可以执行的操作：

#### D9 余额操作

**转账代币**
```javascript
import { ApiPromise, WsProvider } from '@polkadot/api';
import { Keyring } from '@polkadot/keyring';

// 设置
const wsProvider = new WsProvider('ws://127.0.0.1:9944');
const api = await ApiPromise.create({ provider: wsProvider });
const keyring = new Keyring({ type: 'sr25519' });
const sender = keyring.addFromUri('//Alice');

// 转账 100 个代币（根据您的链配置调整小数位数）
const recipient = 'Dn5FHneW52S7C8VmMJgMU34zG8hh9MAYWn1Cv8GfPLscYmMzS';
const amount = 100 * 10**12; // 假设 12 位小数

// 创建并发送交易
const transfer = api.tx.d9Balances.transfer(recipient, amount);

// 订阅交易状态
const unsub = await transfer.signAndSend(sender, ({ status, events }) => {
  if (status.isInBlock) {
    console.log(`交易已包含在区块中：${status.asInBlock}`);

    // 检查成功的转账事件
    events.forEach(({ event }) => {
      if (api.events.d9Balances.Transfer.is(event)) {
        const [from, to, amount] = event.data;
        console.log(`转账：${from} → ${to}: ${amount}`);
      }
    });
  } else if (status.isFinalized) {
    console.log(`交易在区块中最终确定：${status.asFinalized}`);
    unsub(); // 取消订阅
  }
});
```

#### 节点投票操作

**注册为验证节点候选人**
```javascript
// 以 10% 佣金注册为验证节点
const commission = 10; // 10%

const register = api.tx.d9NodeVoting.registerAsCandidate(commission);

await register.signAndSend(account, ({ status }) => {
  if (status.isFinalized) {
    console.log('成功注册为验证节点候选人！');
  }
});
```

**为验证节点投票**
```javascript
// 用您的代币为验证节点投票
const validatorAddress = 'Dn5FHneW52S7C8VmMJgMU34zG8hh9MAYWn1Cv8GfPLscYmMzS';
const voteAmount = 1000 * 10**12; // 1000 个代币

const vote = api.tx.d9NodeVoting.voteForNode(validatorAddress, voteAmount);

await vote.signAndSend(account, ({ status, events }) => {
  if (status.isFinalized) {
    console.log(`为验证节点 ${validatorAddress} 投票 ${voteAmount}`);

    // 检查 VoteCast 事件
    events.forEach(({ event }) => {
      if (api.events.d9NodeVoting.VoteCast.is(event)) {
        console.log('投票成功记录！');
      }
    });
  }
});
```

**移除对验证节点的投票**
```javascript
// 移除您对验证节点的投票
const removeVote = api.tx.d9NodeVoting.removeVote(validatorAddress);

await removeVote.signAndSend(account, ({ status }) => {
  if (status.isFinalized) {
    console.log('成功移除投票');
  }
});
```

**更新验证节点佣金**
```javascript
// 作为验证节点更新您的佣金率
const newCommission = 15; // 15%

const updateCommission = api.tx.d9NodeVoting.updateCommission(newCommission);

await updateCommission.signAndSend(account, ({ status }) => {
  if (status.isFinalized) {
    console.log(`佣金已更新为 ${newCommission}%`);
  }
});
```

#### 查询操作（只读）

这些操作不需要签名或费用：

**查询账户余额**
```javascript
// 获取账户余额
const accountInfo = await api.query.system.account(address);
console.log(`可用余额：${accountInfo.data.free.toHuman()}`);
console.log(`保留金额：${accountInfo.data.reserved.toHuman()}`);
console.log(`Nonce：${accountInfo.nonce}`);
```

**查询推荐关系**
```javascript
// 获取账户的父级（推荐人）
const parent = await api.query.d9Referral.referralRelationships(address);

if (parent.isSome) {
  console.log(`推荐人：${parent.unwrap()}`);
} else {
  console.log('无推荐人（根账户）');
}

// 获取推荐统计
const directReferrals = await api.query.d9Referral.directReferralCount(address);
console.log(`直接推荐数：${directReferrals}`);
```

**查询验证节点投票**
```javascript
// 获取验证节点的总票数
const votes = await api.query.d9NodeVoting.candidateVotes(validatorAddress);
console.log(`总票数：${votes.toHuman()}`);

// 获取验证节点佣金率
const commission = await api.query.d9NodeVoting.validatorCommission(validatorAddress);
console.log(`佣金：${commission}%`);

// 获取您的投票权益
const myVotes = await api.query.d9NodeVoting.votingInterests(myAddress);
console.log('我的投票：', myVotes.toHuman());
```

**订阅新区块**
```javascript
// 订阅新区块头
const unsubscribe = await api.rpc.chain.subscribeNewHeads((header) => {
  console.log(`新区块 #${header.number}: ${header.hash}`);
});

// 稍后：unsubscribe()
```

**订阅事件**
```javascript
// 订阅所有系统事件
api.query.system.events((events) => {
  events.forEach((record) => {
    const { event } = record;

    // 过滤特定事件
    if (api.events.d9Balances.Transfer.is(event)) {
      const [from, to, amount] = event.data;
      console.log(`转账：${from} → ${to}: ${amount.toHuman()}`);
    }

    if (api.events.d9NodeVoting.VoteCast.is(event)) {
      const [voter, candidate, amount] = event.data;
      console.log(`投票：${voter} → ${candidate}: ${amount.toHuman()}`);
    }
  });
});
```

## 理解链规范（创世配置）

### 什么是链规范？

链规范（chain spec）是您区块链的**创世配置**。它定义了：
- **初始状态**：账户余额、验证节点集、治理参数
- **Runtime**：区块链逻辑（WASM 字节码）
- **网络 ID**：链的唯一标识符
- **共识规则**：区块时间、epoch 长度等

将其视为您区块链的**"出生证明"** - 它在所有节点上必须相同，否则它们无法连接。

### 纯文本与原始格式

**纯文本格式**（`testnet-spec-plain.json`）：
- 人类可读的 JSON
- 您可以直接编辑
- 包含清晰的字段名称
- 使用前必须转换为原始格式

**原始格式**（`testnet-spec.json`）：
- 编码/哈希值
- 由实际节点使用
- 不能直接编辑
- 从纯文本格式生成

**工作流程**：编辑纯文本 → 转换为原始格式 → 分发原始格式到所有节点

### 完整工作流程：从助记词到运行中的验证节点

以下是可视化的完整工作流程：

```
┌─────────────────────────────────────────────────────────────────┐
│                    验证节点设置工作流程                         │
└─────────────────────────────────────────────────────────────────┘

步骤 1：生成助记词（在安全机器上）
┌──────────────────────────────────────────────────────────────┐
│ ./d9-node key generate --scheme Sr25519 --words 12          │
│                                                              │
│ 输出："word1 word2 word3 ... word12"                        │
│ 密钥种子：0xabc123...                                       │
│ 公钥（SS58）：Dn5FLSig...                                   │
└──────────────────────────────────────────────────────────────┘
         │
         │ 为每个验证节点重复（SEED1、SEED2、SEED3）
         ↓

步骤 2：派生公钥（用于链规范）
┌──────────────────────────────────────────────────────────────┐
│ # 对于每个助记词，获取三个公钥：                             │
│                                                              │
│ Aura:       ./d9-node key inspect --scheme Sr25519 "$SEED"  │
│ Grandpa:    ./d9-node key inspect --scheme Ed25519 \        │
│               "$SEED//grandpa"                               │
│ ImOnline:   ./d9-node key inspect --scheme Sr25519 \        │
│               "$SEED//im_online"                             │
│                                                              │
│ 账户:      ./d9-node key inspect --network reynolds "$SEED" │
└──────────────────────────────────────────────────────────────┘
         │
         │ 将公钥复制到记事本
         ↓

步骤 3：创建链规范
┌──────────────────────────────────────────────────────────────┐
│ # 生成基础规范                                               │
│ ./d9-node build-spec --chain testnet \                      │
│   --disable-default-bootnode > testnet-plain.json           │
│                                                              │
│ # 编辑 testnet-plain.json：                                 │
│   - 添加余额                                                 │
│   - 使用步骤 2 的公钥添加 session.keys                      │
│   - 设置创世验证节点                                         │
│                                                              │
│ # 转换为原始格式                                             │
│ ./d9-node build-spec --chain testnet-plain.json \           │
│   --raw > testnet.json                                      │
└──────────────────────────────────────────────────────────────┘
         │
         │ 将 testnet.json 分发到所有节点
         ↓

步骤 4：将链规范部署到所有节点
┌──────────────────────────────────────────────────────────────┐
│ scp testnet.json user@validator1:/usr/local/bin/            │
│ scp testnet.json user@validator2:/usr/local/bin/            │
│ scp testnet.json user@validator3:/usr/local/bin/            │
│ scp testnet.json user@bootnode:/usr/local/bin/              │
└──────────────────────────────────────────────────────────────┘
         │
         ↓

步骤 5：将私钥插入验证节点（关键！）
┌──────────────────────────────────────────────────────────────┐
│ # 在验证节点 1 上（使用 SEED1）                             │
│ ./d9-node key insert --base-path /home/ubuntu/node-data \   │
│   --chain /usr/local/bin/testnet.json \                     │
│   --scheme Sr25519 --suri "$SEED1" --key-type aura          │
│                                                              │
│ ./d9-node key insert --base-path /home/ubuntu/node-data \   │
│   --chain /usr/local/bin/testnet.json \                     │
│   --scheme Ed25519 --suri "$SEED1//grandpa" --key-type gran │
│                                                              │
│ ./d9-node key insert --base-path /home/ubuntu/node-data \   │
│   --chain /usr/local/bin/testnet.json \                     │
│   --scheme Sr25519 --suri "$SEED1//im_online" \             │
│   --key-type imon                                            │
│                                                              │
│ # 在验证节点 2 上使用 SEED2 重复                            │
│ # 在验证节点 3 上使用 SEED3 重复                            │
└──────────────────────────────────────────────────────────────┘
         │
         │ 密钥现在在 /home/ubuntu/node-data/chains/*/keystore/
         ↓

步骤 6：验证密钥与链规范匹配
┌──────────────────────────────────────────────────────────────┐
│ # 在每个验证节点上，检查密钥库                               │
│ ls -la /home/ubuntu/node-data/chains/*/keystore/            │
│                                                              │
│ 应该看到 3 个文件：                                          │
│   617572...     (aura - Sr25519)                             │
│   6772616e...   (grandpa - Ed25519)                          │
│   696d6f6e...   (imon - Sr25519)                             │
│                                                              │
│ # 验证公钥与链规范匹配                                       │
│ cat testnet.json | jq '.genesis.runtime.session.keys'       │
└──────────────────────────────────────────────────────────────┘
         │
         ↓

步骤 7：启动节点
┌──────────────────────────────────────────────────────────────┐
│ # 首先启动引导节点                                           │
│ systemctl start d9-node                                      │
│                                                              │
│ # 从日志获取引导节点对等 ID                                 │
│ journalctl -u d9-node -n 50 | grep "Local node identity"    │
│                                                              │
│ # 使用引导节点地址启动验证节点                               │
│ systemctl start d9-node                                      │
└──────────────────────────────────────────────────────────────┘
         │
         ↓

步骤 8：验证网络正在运行
┌──────────────────────────────────────────────────────────────┐
│ # 检查每个节点上的日志                                       │
│ journalctl -u d9-node -f                                     │
│                                                              │
│ 寻找：                                                       │
│   ✓ "Discovered new external address"                       │
│   ✓ "Syncing, target=#X (3 peers)"                          │
│   ✓ "Prepared block for proposing"                          │
│   ✓ "Imported #1"                                            │
│   ✓ "Finalized #1"                                           │
└──────────────────────────────────────────────────────────────┘

成功：网络正在生成和最终确定区块！🎉
```

### 链规范的结构

纯文本链规范包含以下内容：

```json
{
  "name": "D9 Testnet",
  "id": "d9_testnet",
  "chainType": "Live",
  "bootNodes": [],
  "telemetryEndpoints": null,
  "protocolId": "d9",
  "properties": {
    "tokenSymbol": "D9",
    "tokenDecimals": 12,
    "ss58Format": 9
  },
  "genesis": {
    "runtime": {
      // 每个 pallet 的初始配置
      "system": {},
      "d9Balances": {
        "balances": [
          // 预充值账户
        ]
      },
      "session": {
        "keys": [
          // 初始验证节点会话密钥
        ]
      },
      // ... 更多 pallet
    }
  }
}
```

### 自定义您的链规范

#### 1. 设置链身份

```json
{
  "name": "My D9 Testnet",
  "id": "my_d9_testnet",
  "chainType": "Live",  // 或 "Development"、"Local"
  "protocolId": "my-d9"
}
```

**重要**：
- `id` 必须唯一（具有不同 ID 的节点无法连接）
- `protocolId` 应与您的网络名称匹配

#### 2. 配置代币属性

```json
"properties": {
  "tokenSymbol": "D9T",  // 您的代币符号
  "tokenDecimals": 12,    // 小数位数
  "ss58Format": 9         // 地址格式（9 = 地址以 "Dn" 开头）
}
```

#### 3. 添加初始余额

为测试预充值账户：

```json
"d9Balances": {
  "balances": [
    ["Dn5FHneW52S7C8VmMJgMU34zG8hh9MAYWn1Cv8GfPLscYmMzS", 1000000000000000],
    ["Dn5GNW8sW9R9Z7K3xH8YmXq4R7vZ1pJ8xC4vK9mN2sT5eP3aQ", 500000000000000]
  ]
}
```

**注意**：余额金额以最小单位计（使用 12 位小数，所以 1000000000000000 = 1000 个代币）

#### 4. 配置初始验证节点（关键）

您的初始验证节点必须在链规范中才能启动网络：

```json
"session": {
  "keys": [
    [
      "Dn5FHneW52...",  // 验证节点账户（stash）
      "Dn5FHneW52...",  // 验证节点账户（controller - 测试网与 stash 相同）
      {
        "aura": "5GrwvaEF...",      // Aura 公钥（Sr25519）
        "grandpa": "5FA9nQDV...",   // Grandpa 公钥（Ed25519）
        "im_online": "5GNJqTP..."   // ImOnline 公钥（Sr25519）
      }
    ],
    // 添加验证节点 2
    [
      "Dn5GNW8sW9...",
      "Dn5GNW8sW9...",
      {
        "aura": "5HpG9w8E...",
        "grandpa": "5GvvtdD1...",
        "im_online": "5EfLKF8..."
      }
    ],
    // 添加验证节点 3（最少需要 3 个才能使网络运行！）
    [
      "Dn5DqM3nP7...",
      "Dn5DqM3nP7...",
      {
        "aura": "5FLSigC9...",
        "grandpa": "5DFBuDd5...",
        "im_online": "5FoLWcX..."
      }
    ]
  ]
}
```

**如何获取这些值：**

```bash
# 对于每个验证节点，生成密钥并获取公钥
SEED_PHRASE="your twelve word seed phrase here"

# 获取 Aura 公钥
./target/release/d9-node key inspect --scheme Sr25519 "$SEED_PHRASE"

# 获取 Grandpa 公钥
./target/release/d9-node key inspect --scheme Ed25519 "$SEED_PHRASE//grandpa"

# 获取 ImOnline 公钥
./target/release/d9-node key inspect --scheme Sr25519 "$SEED_PHRASE//im_online"

# 获取账户地址（用于 stash/controller）
./target/release/d9-node key inspect --network reynolds "$SEED_PHRASE"
```

### 常见链规范错误

#### ❌ **错误 1：不同节点上的链规范不同**
```
错误："Verification failed: Execution failed: Consensus"
```
**修复**：确保所有节点上的原始链规范完全相同（相同文件、相同哈希）

#### ❌ **错误 2：忘记将纯文本转换为原始格式**
```
错误："Failed to decode chain spec"
```
**修复**：分发前始终使用 `build-spec --raw`

#### ❌ **错误 3：链规范中少于 3 个验证节点**
```
网络启动但从不最终确定区块
```
**修复**：在 `session.keys` 中添加最少 3 个验证节点

#### ❌ **错误 4：错误的 SS58 格式**
```
地址看起来错误或不以 "Dn" 开头
```
**修复**：在属性中设置 `ss58Format: 9`

#### ❌ **错误 5：验证节点密钥与链规范不匹配**
```
验证节点不生成区块
```
**修复**：确保密钥库密钥与链规范中的公钥匹配

#### ❌ **错误 6：忘记链规范中的引导节点**
```
节点无法相互发现
```
**修复**：将引导节点多地址添加到链规范，或使用 `--bootnodes` 标志

## 常见陷阱和常见问题解答

本节提供了团队在部署 D9 测试网时遇到的最常见问题的快速答案。

### 网络无法生成区块

**症状**：网络启动，节点连接，但没有生成区块（`best: #0`）。

**快速诊断**：
```bash
# 检查验证节点是否有会话密钥
ls -la /home/ubuntu/node-data/chains/*/keystore/
# 应该看到以以下开头的文件：617572（aura）、6772616e（grandpa）、696d6f6e（imon）

# 检查正在运行的验证节点数量
# 您需要最少 3 个验证节点才能生成区块
```

**常见原因和解决方案**：

1. **验证节点不足**
   - **问题**：运行的验证节点少于 3 个
   - **修复**：启动至少 3 个具有正确会话密钥的验证节点
   - **原因**：Grandpa 最终确定需要 2/3 + 1 验证节点（最少 3 个中的 2 个）

2. **缺少会话密钥**
   - **问题**：验证节点正在运行但未插入密钥
   - **检查**：`ls /home/ubuntu/node-data/chains/*/keystore/` 显示为空或不完整
   - **修复**：在每个验证节点上插入全部三个所需密钥（aura、grandpa、imon）
   ```bash
   # 每个验证节点需要全部三种密钥类型：
   ./target/release/d9-node key insert --base-path /home/ubuntu/node-data \
     --chain /usr/local/bin/testnet-spec.json --scheme Sr25519 \
     --suri "your seed" --key-type aura

   ./target/release/d9-node key insert --base-path /home/ubuntu/node-data \
     --chain /usr/local/bin/testnet-spec.json --scheme Ed25519 \
     --suri "your seed//grandpa" --key-type gran

   ./target/release/d9-node key insert --base-path /home/ubuntu/node-data \
     --chain /usr/local/bin/testnet-spec.json --scheme Sr25519 \
     --suri "your seed//im_online" --key-type imon

   # 插入密钥后重启
   sudo systemctl restart d9-node.service
   ```

3. **错误的链规范**
   - **问题**：插入密钥时使用的 `--chain` 与运行时不同
   - **检查**：确保所有地方使用相同的链规范文件
   - **修复**：使用正确的链规范路径重新插入密钥

4. **验证节点不在创世块中**
   - **问题**：验证节点地址不在链规范的初始权威中
   - **修复**：在 `palletSession.keys` 中使用正确的验证节点地址重新生成链规范

---

### 验证节点无法最终确定区块

**症状**：区块被生成（`best: #123`）但未被最终确定（`finalized: #0`）。

**快速诊断**：
```bash
# 检查日志中的 Grandpa 消息
journalctl -u d9-node.service -n 100 | grep -i grandpa

# 寻找："Authority set" 或 "Finalizing" 消息
```

**常见原因和解决方案**：

1. **Grandpa 密钥缺失或方案错误**
   - **问题**：Grandpa 密钥必须使用 Ed25519（而不是 Sr25519）
   - **检查**：密钥库应该有以 `6772616e`（"gran" 的十六进制）开头的文件
   - **修复**：
   ```bash
   # 正确：Grandpa 使用 Ed25519 方案
   ./target/release/d9-node key insert \
     --scheme Ed25519 \
     --suri "your seed//grandpa" \
     --key-type gran \
     --base-path /home/ubuntu/node-data \
     --chain /usr/local/bin/testnet-spec.json
   ```

2. **少于 3 个验证节点**
   - **问题**：Grandpa 需要 2/3 超级多数（最少 3 个验证节点）
   - **修复**：部署至少 3 个验证节点

3. **网络分区**
   - **问题**：验证节点无法相互通信
   - **检查**：`system_peers` 应显示验证节点之间的连接
   - **修复**：验证防火墙允许所有验证节点之间的 P2P 端口（40100）

---

### 对等节点无法连接

**症状**：节点显示 `0 peers` 或无法发现网络。

**快速诊断**：
```bash
# 检查系统健康
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "system_health"}' \
  http://localhost:40200 | jq

# 应显示："peers": 3 或更多
```

**常见原因和解决方案**：

1. **引导节点地址错误或无法访问**
   - **问题**：引导节点多地址格式无效
   - **正确格式**：`/ip4/1.2.3.4/tcp/40100/p2p/12D3KooW...`
   - **修复**：
     - 获取引导节点对等 ID：`journalctl -u d9-node.service | grep "Local node identity"`
     - 验证 IP 是公共/可访问的
     - 测试连接：`telnet <bootnode-ip> 40100`

2. **防火墙阻止 P2P 端口**
   - **问题**：端口 40100 未开放
   - **修复**：
   ```bash
   # 在所有节点上
   sudo ufw allow 40100/tcp
   sudo ufw reload

   # 从另一个节点测试
   telnet <node-ip> 40100
   ```

3. **不同的链规范**
   - **问题**：节点使用不同的创世块
   - **检查**：比较所有节点上的链规范哈希：
   ```bash
   md5sum /usr/local/bin/testnet-spec.json
   # 哈希在所有节点上必须匹配
   ```
   - **修复**：将完全相同的链规范文件复制到所有节点

4. **NAT/端口转发问题**
   - **问题**：NAT 后面的节点无法接受传入连接
   - **修复**：在路由器上配置端口转发或使用 `--public-addr`：
   ```bash
   --public-addr /ip4/<PUBLIC_IP>/tcp/40100
   ```

---

### 交易卡在池中

**症状**：交易已提交但从未包含在区块中。

**快速诊断**：
```bash
# 检查交易池
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "author_pendingExtrinsics"}' \
  http://localhost:40200 | jq
```

**常见原因和解决方案**：

1. **费用余额不足**
   - **问题**：账户有余额但不足以支付交易费用
   - **修复**：确保账户至少有额外的 0.01 D9 用于费用
   ```javascript
   // 检查可用和保留余额
   const { data: { free, reserved } } = await api.query.system.account(address);
   console.log(`可用：${free}，保留：${reserved}`);
   ```

2. **无效的 Nonce**
   - **问题**：Nonce 不按顺序（有间隙或重复）
   - **检查**：比较账户 nonce 与交易 nonce
   ```javascript
   const { nonce } = await api.query.system.account(address);
   console.log(`当前 nonce：${nonce}`);
   // 下一个交易必须使用此 nonce
   ```
   - **修复**：使用正确的 nonce 重新提交

3. **交易被丢弃（低优先级）**
   - **问题**：池已满，交易优先级低
   - **修复**：增加小费：
   ```javascript
   await api.tx.balances.transfer(recipient, amount)
     .signAndSend(sender, { tip: 1000000000 }); // 添加小费
   ```

4. **节点未同步**
   - **问题**：提交到未赶上的节点
   - **检查**：`best` 应等于 `finalized`（±1-2 个区块）
   - **修复**：等待同步或连接到已同步的节点

---

### 密钥无法被识别

**症状**：已插入验证节点密钥但节点不使用它们进行区块生成。

**快速诊断**：
```bash
# 列出密钥库文件
ls -l /home/ubuntu/node-data/chains/*/keystore/

# 每个验证节点应看到 3 个文件：
# 617572...（aura - Sr25519）
# 6772616e...（grandpa - Ed25519）
# 696d6f6e...（imon - Sr25519）
```

**常见原因和解决方案**：

1. **错误的密钥方案**
   - **问题**：为密钥类型使用了错误的加密方案
   - **正确方案**：
     - Aura：**Sr25519**
     - Grandpa：**Ed25519** ⚠️（常见错误 - 人们使用 Sr25519）
     - ImOnline：**Sr25519**
   - **修复**：删除错误的密钥并使用正确的方案重新插入

2. **密钥库路径不匹配**
   - **问题**：密钥插入到的 `--base-path` 与运行时使用的不同
   - **检查**：服务文件 `--base-path` 与密钥插入命令匹配
   - **修复**：确保两者使用相同的路径（例如，`/home/ubuntu/node-data`）

3. **链规范不匹配**
   - **问题**：密钥插入时使用的 `--chain` 与服务使用的不同
   - **修复**：
   ```bash
   # 服务文件和密钥插入必须使用相同的链规范
   # 服务：--chain /usr/local/bin/testnet-spec.json
   # 密钥：--chain /usr/local/bin/testnet-spec.json
   ```

4. **插入后未重启节点**
   - **问题**：已插入密钥但未重启节点
   - **修复**：
   ```bash
   sudo systemctl restart d9-node.service
   journalctl -u d9-node.service -f
   # 查找 "Loaded block-authoring keys" 消息
   ```

5. **文件权限**
   - **问题**：节点进程无法读取密钥库文件
   - **修复**：
   ```bash
   sudo chown -R ubuntu:ubuntu /home/ubuntu/node-data
   chmod 600 /home/ubuntu/node-data/chains/*/keystore/*
   ```

---

### RPC/WebSocket 连接失败

**症状**：无法通过 Polkadot.js 或自定义应用程序连接到 RPC 节点。

**快速诊断**：
```bash
# 测试 RPC 端点
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "system_health"}' \
  http://<node-ip>:40200

# 测试 WebSocket（从浏览器控制台）
const ws = new WebSocket('ws://<node-ip>:9944');
ws.onopen = () => console.log('已连接！');
ws.onerror = (e) => console.error('错误：', e);
```

**常见原因和解决方案**：

1. **RPC 未对外暴露**
   - **问题**：节点仅监听本地主机（127.0.0.1）
   - **修复**：向 systemd 服务添加标志：
   ```ini
   --rpc-external \
   --rpc-cors all \
   --ws-external
   ```
   ⚠️ **安全**：仅在专用 RPC 节点上暴露 RPC/WS，不要在验证节点上

2. **防火墙阻止端口**
   - **问题**：端口 40200（RPC）和 9944（WS）未开放
   - **修复**：
   ```bash
   sudo ufw allow 40200/tcp
   sudo ufw allow 9944/tcp
   sudo ufw reload
   ```

3. **CORS 问题**
   - **问题**：浏览器由于 CORS 策略阻止连接
   - **症状**：控制台显示 "CORS policy" 错误
   - **修复**：添加 `--rpc-cors all` 或特定来源：
   ```bash
   --rpc-cors "https://polkadot.js.org,http://localhost:3000"
   ```

4. **连接字符串中的端口错误**
   - **问题**：连接到错误的端口
   - **默认值**：RPC=9933，自定义 RPC=40200，WS=9944
   - **修复**：匹配服务配置：
   ```javascript
   // 如果服务使用 --rpc-port 40200
   const api = await ApiPromise.create({
     provider: new WsProvider('ws://node-ip:9944')  // WS 使用默认 9944
   });
   ```

5. **WSS 的 SSL/TLS 问题**
   - **问题**：在没有 SSL 证书的情况下使用 `wss://`
   - **修复**：选择：
     - 测试时使用 `ws://`（不安全）
     - 为生产 `wss://` 设置 Nginx 和 Let's Encrypt

---

### FAQ：快速参考

**问：引导节点需要验证节点密钥吗？**
**答：**❌ **不需要**。引导节点仅帮助对等发现。只有验证节点需要 Aura、Grandpa 和 ImOnline 密钥。

---

**问：正常工作的网络最少需要多少验证节点？**
**答：**最少 **3 个验证节点**。Grandpa 最终确定需要 2/3 超级多数，这意味着 3 个验证节点中的 2 个（2/3 + 1 = 3）。

---

**问：我可以在同一节点上运行验证节点和 RPC 吗？**
**答：**技术上可以，但**不推荐**。安全最佳实践：验证节点不应公开暴露 RPC/WS。使用专用 RPC 节点。

---

**问：`best` 和 `finalized` 区块有什么区别？**
**答：**
- `best`：最新生成的区块（Aura）
- `finalized`：由 2/3+ 验证节点确认的区块（Grandpa）
- 通常 `finalized` 比 `best` 滞后 1-3 个区块

---

**问：为什么用户无法向我的验证节点发送交易？**
**答：**验证节点使用 P2P 端口（40100）仅用于共识。用户需要 RPC/WebSocket（40200/9944），这应该只在专用 RPC 节点上暴露，而不是验证节点。

---

**问：我需要为每种密钥类型使用单独的助记词吗？**
**答：**不需要。使用相同的助记词，派生不同的密钥：
- Aura：`seed`
- Grandpa：`seed//grandpa`
- ImOnline：`seed//im_online`

---

**问：我如何知道我的验证节点是否正在积极生成区块？**
**答：**检查日志中的 "Starting consensus session"：
```bash
journalctl -u d9-node.service -f | grep "consensus\|Prepared block"
```

---

**问：网络启动后我可以更改验证节点密钥吗？**
**答：**可以，但需要会话轮换：
1. 插入新密钥
2. 调用 `session.setKeys()` 外部调用
3. 等待会话更改（~1 个 era）

---

**问：如果验证节点离线会发生什么？**
**答：**
- 网络继续使用剩余的验证节点（如果 ≥3 仍在线）
- 离线验证节点错过区块生成槽
- 可能因停机时间被削减（检查运行时配置）

---

**问：我的节点显示"Idle" - 这正常吗？**
**答：**取决于：
- **引导节点/RPC 节点**：是的，正常（它们不生成区块）
- **验证节点**：否，检查会话密钥是否已插入且网络有 ≥3 个验证节点

---

**问：初始同步需要多长时间？**
**答：**
- **新测试网**：几分钟（小链）
- **主网**：几小时到几天，取决于链大小
- 检查进度：`best` 应接近 `finalized`

---

## 安全考虑

### 助记词安全

- **永远不要**通过不安全的渠道分享助记词
- 将助记词存储在加密的密码管理器中
- 在安全的物理位置保留离线备份
- 为测试网和主网使用不同的助记词

### 网络安全

```bash
# 配置防火墙（使用 ufw 的示例）
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 40100/tcp  # P2P
# 如果需要，仅允许来自特定 IP 的 RPC
sudo ufw allow from <YOUR_IP> to any port 40200
sudo ufw enable
```

### 服务安全

```bash
# 以非 root 用户运行节点（已在 systemd 中配置）
# 限制文件权限
chmod 700 /home/ubuntu/node-data
chmod 600 /home/ubuntu/node-data/chains/*/keystore/*
```

## 下一步

部署测试网后：

1. **监控性能**：在前 24 小时观察日志和资源使用情况
2. **测试功能**：尝试交易、质押和治理功能
3. **记录问题**：记录任何问题以供主网部署参考
4. **计划升级**：首先在测试网上测试运行时升级程序
5. **备份密钥**：确保所有验证节点密钥都已安全备份

## 资源

- [D9 节点文档](../README.md)
- [安装指南](./installation.md)
- [运行节点](./running-a-node.md)
- [GitHub 仓库](https://github.com/D-Nine-Chain/d9-node)
- [Discord 社区](https://discord.gg/d9chain)

## 支持

如需测试网部署帮助：

- [GitHub Issues](https://github.com/D-Nine-Chain/d9-node/issues)
- [Discord 社区](https://discord.gg/d9chain)

---

*本文档由 D9 团队维护 - 最后更新：2025 年*

