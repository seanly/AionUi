# OpenClaw Gateway 配置指南

## 概述

AionUi 通过 WebSocket 连接 OpenClaw Gateway。Gateway 可以在本地运行（自动启动），也可以部署在远程服务器上。

配置文件路径：`~/.openclaw/openclaw.json`（支持 JSONC 注释语法）。

## 认证流程

OpenClaw Gateway 采用**基于设备的认证**模型：

1. **共享 Token/密码** — 在 `openclaw.json` 中配置，用于首次连接认证
2. **设备身份** — AionUi 自动生成密钥对（存储在 `~/.openclaw/`），每次连接时用私钥签名
3. **设备 Token** — 首次认证成功后，Gateway 返回设备专属 Token，本地存储，后续连接优先使用

`openclaw.json` 中的 token 是**服务端设置的共享密钥**，所有客户端使用同一个 token 做首次认证。握手完成后，每个设备会获得自己的专属 token。

```
首次连接：共享 token + 设备签名 → hello-ok + 设备 token
后续连接：设备 token + 设备签名 → hello-ok（+ 刷新）
```

## 配置示例

### 示例 1：本地 Gateway（零配置）

无需配置文件，AionUi 自动启动本地 Gateway 进程。

```
结果：
  url      = ws://localhost:18789
  external = false → 自动启动 Gateway
  auth     = 无
```

### 示例 2：本地 Gateway 自定义端口

```json
{
  "gateway": {
    "port": 9999
  }
}
```

```
结果：
  url      = ws://localhost:9999
  external = false → 自动启动或检测已有进程
  auth     = 无
```

### 示例 3：通过 URL 连接远程服务器

```json
{
  "gateway": {
    "url": "ws://192.168.1.100:18789",
    "auth": {
      "mode": "token",
      "token": "your-shared-token"
    }
  }
}
```

```
结果：
  url      = ws://192.168.1.100:18789
  external = true（从 url 自动推断）
  auth     = token
```

### 示例 4：通过 host + port 连接远程服务器

```json
{
  "gateway": {
    "host": "192.168.1.100",
    "port": 18789,
    "auth": {
      "mode": "token",
      "token": "your-shared-token"
    }
  }
}
```

```
结果：
  url      = ws://192.168.1.100:18789
  external = true（从远程 host 自动推断）
  auth     = token
```

### 示例 5：加密连接 (wss://)

```json
{
  "gateway": {
    "url": "wss://ai.example.com",
    "auth": {
      "mode": "token",
      "token": "your-shared-token"
    }
  }
}
```

```
结果：
  url      = wss://ai.example.com
  external = true
  auth     = token（TLS 加密传输）
```

### 示例 6：密码认证

```json
{
  "gateway": {
    "url": "ws://192.168.1.100:18789",
    "auth": {
      "mode": "password",
      "password": "your-password"
    }
  }
}
```

## 配置参考

### `gateway` 对象

| 字段   | 类型   | 默认值      | 说明                                     |
| ------ | ------ | ----------- | ---------------------------------------- |
| `url`  | string | —           | 完整 WebSocket URL，优先于 `host`/`port` |
| `host` | string | `localhost` | Gateway 主机名或 IP                      |
| `port` | number | `18789`     | Gateway 端口                             |
| `auth` | object | —           | 认证配置                                 |

### `gateway.auth` 对象

| 字段       | 类型   | 可选值                      | 说明                     |
| ---------- | ------ | --------------------------- | ------------------------ |
| `mode`     | string | `none`, `token`, `password` | 认证方式                 |
| `token`    | string | —                           | 共享 token（mode=token） |
| `password` | string | —                           | 密码（mode=password）    |

## 配置值优先级

每个配置值按以下顺序解析（取第一个非空值）：

```
url:      程序传入 → openclaw.json gateway.url  → ws://{host}:{port}
host:     程序传入 → openclaw.json gateway.host → "localhost"
port:     程序传入 → openclaw.json gateway.port → 18789
token:    程序传入 → openclaw.json gateway.auth.token    → （无）
password: 程序传入 → openclaw.json gateway.auth.password → （无）
```

## 外部 Gateway 自动推断

当未显式设置 `useExternalGateway` 时，按以下规则自动推断：

| 条件                                  | `useExternal`                  |
| ------------------------------------- | ------------------------------ |
| 配置了 `url`（程序传入或配置文件）    | `true`                         |
| `host` 不是 `localhost` / `127.0.0.1` | `true`                         |
| 其他情况                              | `false` → 尝试启动本地 Gateway |

当 `useExternalGateway` 显式设为 `false` 时，配置文件中的远程 `url` 会被忽略，连接始终使用 `ws://{host}:{port}`。

## 服务端部署

在远程服务器上启动 Gateway：

```bash
# 无认证
openclaw gateway --port 18789

# Token 认证
openclaw gateway --port 18789 --auth-token "your-shared-token"

# 密码认证
openclaw gateway --port 18789 --auth-password "your-password"
```

共享 token/密码在启动 Gateway 时设置，所有客户端使用相同的共享凭证进行首次认证。
