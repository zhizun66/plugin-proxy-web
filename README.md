# King Box 代理池插件（`proxy_desktop` / `proxy_web`）

**代理池（Proxy Pool）** 是 King Box 生态的核心网络基础设施双轨插件，遵循「万物皆插件」架构设计，同时提供 **Desktop 本地客户端插件（`proxy_desktop`）** 与 **Web 多租户云端控制面插件（`proxy_web`）**。

插件不仅提供可视化的代理节点管理、智能批量导入、并发连通性及出口归属地测通、渠道账号占用监控，还通过宿主服务网关向其他业务插件（如 `discord_desktop` 等）暴露标准的 `proxy.v1` 自动选座与绑定治理服务，并支持 Desktop 与 Web 控制面之间的实时双向同步。

---

## 核心功能特性

### 1. 代理节点全生命周期管理
- **多协议与明文凭据展示**：完整支持 `HTTP` 与 `SOCKS5` 代理协议，支持用户名/密码认证与无认证节点；代理列表「`主机:端口:账号:密码`」列直接展示完整连接串并提供一键复制按钮（右下角气泡提示），编辑弹窗明文回显并支持直接修改或清空密码。
- **原地编辑与跨端重命名收敛**：修改已有代理的 `IP / 主机`、`端口`、`协议` 或 `用户名` 时，双端基于 `previous_keys` 原地更新记录并自动迁移活动占用绑定，不会重复新建记录或额外占用配额。
- **智能批量导入**：内置多格式解析器，支持粘贴单行或多行文本一键识别导入，兼容：
  - 标准 URL 格式：`socks5://user:pass@host:port`、`http://host:port`
  - 四段/三段/两段冒号分隔：`host:port:user:pass`、`host:port:user`、`host:port`
  - 自动按 `scheme + host + port + username` 四元组（`ProxyKey`）去重，避免重复入库。
- **并发测通与出口地区识别**：
  - 支持单条、多选或全量代理并发测通（默认 4 并发协程池，可随时中断停止）；
  - 自动探测代理连通延迟（`check_ms`）并识别出口 IP 所属国家/地区、省份与城市（如 `US · California · Los Angeles`），实时推送进度刷新列表。

### 2. 跨插件代理自动选座服务（`proxy.v1`）
面向宿主内其他业务插件（如 Discord、Telegram 等矩阵插件）提供统一的 `proxy`（`v1`）RPC 服务：
- **最少占用优先选座（`proxy.assign` / `proxy.replace` / `proxy.rotate`）**：按渠道（`channel`）统计各代理节点当前活跃绑定数，优先分配可用（`status=ok`）且当前渠道占用最少的代理节点。
- **幂等绑定与故障轮换**：支持同一渠道账号幂等复用已有座位、指定代理强制换绑（`proxy.bind`）、排除当前异常代理并自动轮换新节点（`proxy.rotate`）。
- **状态上报与释放（`proxy.report` / `proxy.release`）**：业务插件可实时上报账号在当前代理下的健康状态（`ok` / `bad` / `unknown`）或在账号删除时自动释放座位。

### 3. Desktop ⇄ Web 实时双向云同步
- **全自动增量与定时收敛**：
  - Desktop 端新增、编辑、批量导入、删除代理或完成测通后，立即触发实时上行同步；
  - 后台内置防抖指纹检测协程（本地变更 8 秒内触发同步，无变更每 24 秒轮询拉取云端变更），自动拉取合并 Web 端新增/修改/删除的代理节点；
  - 支持在 Desktop 页面点击「同步云端」按钮一键触发全量双向并集对齐（`push_full`）。
- **跨端使用记录监控与远程释放**：
  - Desktop 端各渠道账号的代理占用实时镜像至 Web 控制面「使用记录」页面；
  - 管理员在 Web 端手动释放某条占用后，下次同步将通过 `released_bindings` 指令自动释放 Desktop 本地对应座位。
- **多租户隔离与配额门禁**：
  - Web 端基于 PostgreSQL 独立 Schema（`proxy.proxies` 与 `proxy.proxy_bindings`）按认证上下文可信 `tenant_id` 严格隔离多租户数据；
  - 集成 `gatesdk` 配额门禁（配额键 `proxies`，默认上限 `2000` 条），新增与批量导入时自动校验并校准租户配额。

---

## 目录结构

```text
plugins/proxy/
├── apps/
│   ├── desktop/                 # Desktop 本地插件轨道 (proxy_desktop)
│   │   ├── backend/main.go      # Desktop 端 Go 后端（SQLite + proxy.v1 + 云端双向同步）
│   │   ├── ui/                  # Desktop 端前端页面（代理池管理 + 使用记录）
│   │   └── manifest.json        # Desktop 端插件清单
│   └── web/                     # Web 云端插件轨道 (proxy_web)
│       ├── backend/main.go      # Web 端 Go 后端（PostgreSQL 多租户 + proxy.sync 接口）
│       ├── ui/                  # Web 端前端页面（多租户代理池 + 全局只读视图）
│       └── manifest.json        # Web 端插件清单
├── public/
│   └── backend/                 # 双端共享核心库（模型、GORM 仓储、并发测通器、DTO）
├── assets/
│   └── logo.svg                 # 插件矢量图标（适配亮色/暗色双主题）
├── docs/
│   └── sync-api.md              # Web 端暴露给 Desktop 端插件同步的 API 接口规范
├── Makefile                     # 双端构建、开发联调与多平台打包发布脚本
└── README.md                    # 插件功能与使用说明文档
```

---

## `proxy.v1` 服务操作一览

| 操作名（`operation`） | 支持端 | 说明 |
| :--- | :--- | :--- |
| `list` | Desktop / Web | 分页查询代理池列表（支持关键词、状态筛选与时间排序） |
| `create` / `update` / `delete` | Desktop / Web | 创建、更新、单个或批量删除代理节点 |
| `import` | Desktop / Web | 批量去重导入代理节点并自动校验 `proxies` 配额 |
| `check` / `check.stop` | Desktop / Web | 启动或停止代理并发连通性与出口地区测通任务 |
| `bindings.list` / `binding.release` | Desktop / Web | 分页查询渠道账号代理占用列表、手动释放指定占用 |
| `picker` | Desktop | 查询指定渠道可选代理列表及各节点在该渠道的当前占用数 |
| `proxy.assign` / `proxy.bind` | Desktop | 为渠道账号自动分配最优代理或绑定指定代理 |
| `proxy.replace` / `proxy.rotate` | Desktop | 替换或轮换渠道账号当前占用的代理节点 |
| `proxy.report` / `proxy.release` / `proxy.active` | Desktop | 上报绑定健康状态、释放账号代理占用、查询账号当前生效代理 ID |
| `proxy.push_sync` / `proxy.pull_sync` | Desktop | 手动触发与 Web 控制面的双向全量同步或仅下行拉取 |
| `proxy.sync` | Web | 接收 Desktop 端通过 `license.cloud.invoke` 发起的双向同步请求（详见 [`docs/sync-api.md`](./docs/sync-api.md)） |

---

## 构建与发布指南

在 `plugins/proxy` 目录下可通过 `Makefile` 执行构建与发布：

```bash
# 1. 同时构建 Desktop 与 Web 双端产物（输出至 target/desktop 与 target/web）
make build

# 2. 打包并发布 Web 端多平台安装包与 README.md 至 GitHub Release
make package-web PUBLISH=1

# 3. 打包并发布 Desktop 端安装包与 README.md 至 GitHub Release
make package-desktop PUBLISH=1
```

## 相关仓库

- **源码仓库（私有）**：`http://localhost:3000/root/plugin-proxy.git`
- **Web 端发布仓库（公开）**：`https://github.com/zhizun66/plugin-proxy-web`
- **Desktop 端发布仓库（公开）**：`https://github.com/zhizun66/plugin-proxy-desktop`
