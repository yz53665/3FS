# 3FS 双节点 TCP-Only 部署指南

## 拓扑架构

```
机器 A (192.168.1.10, NodeId=1)         机器 B (192.168.1.20, NodeId=10001)
┌─────────────────────────────┐        ┌─────────────────────────────┐
│ MGMTD  (NodeId=1)           │        │ Storage  (NodeId=10001)     │
│  ├─ TCP:8000 (Mgmtd服务)    │        │  ├─ TCP:8000 (Storage服务)  │
│  └─ TCP:9000 (Core服务)     │        │  └─ TCP:9000 (Core服务)     │
│                             │        │                             │
│ Meta   (NodeId=100)         │        │ FUSE 客户端                 │
│  ├─ TCP:8001 (Meta服务)     │◄──────►│  └─ 挂载 /mnt/3fs          │
│  └─ TCP:9001 (Core服务)     │  TCP   │                             │
│                             │        │                             │
│ Storage (NodeId=10000)      │        │                             │
│  ├─ TCP:8000 (Storage服务)  │        │                             │
│  └─ TCP:9000 (Core服务)     │        │                             │
│                             │        │                             │
│ FUSE 客户端                 │        │                             │
│  └─ 挂载 /mnt/3fs          │        │                             │
└─────────────────────────────┘        └─────────────────────────────┘
```

- **机器 A**：运行 MGMTD、Meta、Storage、FUSE 客户端
- **机器 B**：运行 Storage、FUSE 客户端
- 所有通信通过 **TCP 协议**，无需 RDMA 硬件
- 集群元数据存储使用 **FoundationDB**

---

## 配置文件说明

每个组件使用三个配置文件（位于 `/opt/3fs/config/`）：

| 文件 | 作用 | 核心内容 |
|------|------|----------|
| `*_main.toml` | 通用配置 | 集群标识、客户端网络、`[ib_devices]`、`[mgmtd]`/`[mgmtd_client]`、`[client]` |
| `*_main_launcher.toml` | 启动器配置 | `node_id`、`allow_empty_node_id`（仅此两个变量） |
| `*_main_app.toml` | 应用/服务端配置 | 日志、线程池、监听端口、业务逻辑参数 |

---

## 前置准备

### 1. 编译项目

在两台机器上执行：

```bash
git clone <repo_url> 3FS
cd 3FS
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo
make -j$(nproc)
```

编译产物位于 `build/bin/`，包括：
- `mgmtd_main` — 管理守护进程
- `meta_main` — 元数据服务
- `storage_main` — 存储服务
- `hf3fs_fuse_main` — FUSE 客户端
- `admin_cli` — 管理命令行工具

### 2. 安装 FoundationDB（仅机器 A）

```bash
# 添加 FDB 仓库并安装
# Ubuntu/Debian:
wget https://github.com/apple/foundationdb/releases/download/7.1.64/foundationdb-clients_7.1.64-1_amd64.deb
wget https://github.com/apple/foundationdb/releases/download/7.1.64/foundationdb-server_7.1.64-1_amd64.deb
sudo dpkg -i foundationdb-clients_7.1.64-1_amd64.deb foundationdb-server_7.1.64-1_amd64.deb

# 验证
fdbcli --exec "status minimal"

# 单机内存模式，测试用
fdbcli --exec "configure new single memory"
```

> FoundationDB 只需在机器 A 部署，其他节点通过 MGMTD 的 FDB 客户端连接即可。

### 3. 创建目录结构

```bash
# 两台机器都执行
sudo mkdir -p /opt/3fs/{bin,config,data}
sudo mkdir -p /var/log/3fs
sudo mkdir -p /mnt/3fs

# 复制二进制
sudo cp build/bin/mgmtd_main /opt/3fs/bin/
sudo cp build/bin/meta_main /opt/3fs/bin/
sudo cp build/bin/storage_main /opt/3fs/bin/
sudo cp build/bin/hf3fs_fuse_main /opt/3fs/bin/
sudo cp build/bin/admin_cli /opt/3fs/bin/

# 复制配置文件模板
sudo cp configs/*.toml /opt/3fs/config/
```

### 4. 确保网络互通

```bash
# 机器 A 上 ping 机器 B
ping 192.168.1.20

# 机器 B 上 ping 机器 A
ping 192.168.1.10

# 检查端口未被占用
netstat -tlnp | grep -E '800[0-9]|900[0-9]'
```

---

## 第一步：配置 MGMTD（机器 A）

### 1.1 修改 `/opt/3fs/config/mgmtd_main.toml`

```toml
allow_dev_version = true
cluster_id = 'mycluster'

# FDB 连接配置
[fdb]
clusterFile = '/etc/foundationdb/fdb.cluster'

# 也可以使用 KV 引擎配置方式
[kv_engine]
use_memkv = false
[kv_engine.fdb]
clusterFile = '/etc/foundationdb/fdb.cluster'

# 允许无 RDMA 设备（TCP-only）
[ib_devices]
allow_no_usable_devices = true
```

### 1.2 修改 `/opt/3fs/config/mgmtd_main_launcher.toml`

```toml
allow_empty_node_id = false
node_id = 1            # MGMTD 固定使用 NodeId=1
```

### 1.3 修改 `/opt/3fs/config/mgmtd_main_app.toml`

将 `[[server.base.groups]]` 中的服务组改为 TCP：

```toml
# 第一个服务组：Mgmtd 业务 → TCP
[[server.base.groups]]
services = ['Mgmtd']
network_type = "TCP"                  # 改为 TCP
[server.base.groups.listener]
listen_port = 8000
# ... 其余子配置保持模板不变 ...

# 第二个服务组：Core → TCP
[[server.base.groups]]
services = ['Core']
network_type = "TCP"                  # 改为 TCP
[server.base.groups.listener]
listen_port = 9000
# ... 其余子配置保持模板不变 ...
```

同时添加客户端 TCP 配置：

```toml
[server.client]
force_use_tcp = true
```

### 1.4 启动 MGMTD

```bash
sudo /opt/3fs/bin/mgmtd_main \
    --launcher_config /opt/3fs/config/mgmtd_main_launcher.toml \
    --app_config /opt/3fs/config/mgmtd_main_app.toml \
    --config /opt/3fs/config/mgmtd_main.toml
```

验证：

```bash
# 检查进程
ps aux | grep mgmtd_main

# 检查日志
tail -f /var/log/3fs/mgmtd_main.log
```

---

## 第二步：配置 Meta（机器 A）

### 2.1 修改 `/opt/3fs/config/meta_main.toml`

```toml
allow_dev_version = true
cluster_id = 'mycluster'

# 客户端出站连接走 TCP
[client]
force_use_tcp = true

# 允许无 RDMA 设备
[ib_devices]
allow_no_usable_devices = true

# MGMTD 客户端配置
[mgmtd_client]
mgmtd_server_addresses = ['TCP://192.168.1.10:8000']
enable_auto_heartbeat = true
```

### 2.2 修改 `/opt/3fs/config/meta_main_launcher.toml`

```toml
allow_empty_node_id = false
node_id = 100
```

### 2.3 修改 `/opt/3fs/config/meta_main_app.toml`

服务组改为 TCP：

```toml
# 第一个服务组：MetaSerde → TCP
[[server.base.groups]]
services = ['MetaSerde']
network_type = "TCP"
[server.base.groups.listener]
listen_port = 8001
# ...

# 第二个服务组：Core → TCP
[[server.base.groups]]
services = ['Core']
network_type = "TCP"
[server.base.groups.listener]
listen_port = 9001
# ...
```

添加客户端和 FDB 配置：

```toml
[server.client]
force_use_tcp = true

[server.fdb]
clusterFile = '/etc/foundationdb/fdb.cluster'

[server.kv_engine]
use_memkv = false
[server.kv_engine.fdb]
clusterFile = '/etc/foundationdb/fdb.cluster'
```

### 2.4 启动 Meta

```bash
sudo /opt/3fs/bin/meta_main \
    --launcher_config /opt/3fs/config/meta_main_launcher.toml \
    --app_config /opt/3fs/config/meta_main_app.toml \
    --config /opt/3fs/config/meta_main.toml
```

---

## 第三步：配置 Storage Server（两台机器）

> 注意：Storage 服务端使用 `storage_main_app.toml`（服务端业务配置），
> 而 `storage_main.toml` 用于 FUSE 客户端，此处不需要修改。

### 3.1 机器 A - Storage (NodeId=10000)

**修改 `/opt/3fs/config/storage_main_launcher.toml`**：

```toml
allow_empty_node_id = false
node_id = 10000
```

**修改 `/opt/3fs/config/storage_main_app.toml`**：

服务组改为 TCP：

```toml
# 第一个服务组：StorageSerde → TCP
[[server.base.groups]]
services = ['StorageSerde']
network_type = "TCP"
[server.base.groups.listener]
listen_port = 8000
# ...

# 第二个服务组：Core → TCP
[[server.base.groups]]
services = ['Core']
network_type = "TCP"
[server.base.groups.listener]
listen_port = 9000
# ...
```

客户端和转发客户端走 TCP：

```toml
[server.client]
force_use_tcp = true

[server.forward_client]
force_use_tcp = true
```

设置数据目录：

```toml
[server.targets]
target_paths = ['/opt/3fs/data/storage']
allow_disk_without_uuid = true
```

### 3.2 机器 B - Storage (NodeId=10001)

操作与机器 A 相同，区别：
- `storage_main_launcher.toml`: `node_id = 10001`
- 创建数据目录：`sudo mkdir -p /opt/3fs/data/storage`

### 3.3 启动 Storage

```bash
sudo mkdir -p /opt/3fs/data/storage
sudo /opt/3fs/bin/storage_main \
    --launcher_config /opt/3fs/config/storage_main_launcher.toml \
    --app_config /opt/3fs/config/storage_main_app.toml
```

> 注意：Storage Server 无需 `--config` 参数（不使用 `storage_main.toml`），
> 服务端配置都在 `storage_main_app.toml` 中。

---

## 第四步：创建 Target 和 Chain

在机器 A 上使用 `admin_cli` 初始化集群。

### 4.1 配置 admin_cli

admin_cli 读取的是 FUSE 客户端配置文件 `hf3fs_fuse_main.toml`，
因此先配置好 TCP 连接参数（见第五步）即可使用。

### 4.2 初始化集群

```bash
# 1. 初始化
admin_cli --config /opt/3fs/config/hf3fs_fuse_main.toml \
    "init-cluster --mgmtd 192.168.1.10:8000 --desc '2-node TCP test cluster'"

# 2. 创建管理员用户
admin_cli --config /opt/3fs/config/hf3fs_fuse_main.toml \
    "create-user --root-admin --token admin_token"

# 3. 列出节点，确认所有节点都已上线
admin_cli --config /opt/3fs/config/hf3fs_fuse_main.toml \
    "list-nodes"
```

### 4.3 创建 Storage Target

```bash
# 为机器 A 的 Storage 创建 target
admin_cli --config /opt/3fs/config/hf3fs_fuse_main.toml \
    "create-target --node-id 10000 --target-id 1 --disk-index 0 --chain-id 1"

admin_cli --config /opt/3fs/config/hf3fs_fuse_main.toml \
    "create-target --node-id 10000 --target-id 3 --disk-index 0 --chain-id 2"

# 为机器 B 的 Storage 创建 target
admin_cli --config /opt/3fs/config/hf3fs_fuse_main.toml \
    "create-target --node-id 10001 --target-id 2 --disk-index 0 --chain-id 1"

admin_cli --config /opt/3fs/config/hf3fs_fuse_main.toml \
    "create-target --node-id 10001 --target-id 4 --disk-index 0 --chain-id 2"
```

### 4.4 选择 Chain 配置方案

双机场景有几种 Chain 布局可选，根据测试目标选择。

---

#### 方案 A：双副本单链（简单，一个 Head 有压力）

```
Chain C1: [Target_1(A), Target_2(B)]
         Head=A          Succ=B

所有 Chunk → C1 → 写请求全部打到 A
```

每个 Chunk 2 副本，任意一台宕机数据完好。但所有写入的 Head 都在 A。

```bash
cat > /tmp/chain_table.json << 'EOF'
[
  {
    "chainId": 1,
    "chainVersion": 1,
    "targets": [
      { "targetId": 1, "chainId": 1 },
      { "targetId": 2, "chainId": 1 }
    ]
  }
]
EOF
```

---

#### 方案 B：双副本双链，Head 互换（推荐）

```
Chain C1: [Target_1(A), Target_2(B)]     Chain C2: [Target_2(B), Target_1(A)]
         Head=A          Succ=B                   Head=B          Succ=A

Chunk[偶] → C1 → 写请求打到 A       Chunk[奇] → C2 → 写请求打到 B
```

Head 负载在两台机器间轮转，写入性能更均衡。

```bash
cat > /tmp/chain_table.json << 'EOF'
[
  {
    "chainId": 1,
    "chainVersion": 1,
    "targets": [
      { "targetId": 1, "chainId": 1 },
      { "targetId": 2, "chainId": 1 }
    ]
  },
  {
    "chainId": 2,
    "chainVersion": 1,
    "targets": [
      { "targetId": 2, "chainId": 2 },
      { "targetId": 1, "chainId": 2 }
    ]
  }
]
EOF
```

---

#### 方案 C：单副本双链（仅基准测试，无容错）

```
Chain C1: [Target_1(A)]          Chain C2: [Target_2(B)]
         单副本                            单副本

Chunk[偶] → C1 → 只存 A         Chunk[奇] → C2 → 只存 B
```

零转发开销，写延迟最低。但任意一台宕机丢失一半 Chunk。**仅用于 NDS 直通延迟基准测试。**

```bash
cat > /tmp/chain_table.json << 'EOF'
[
  {
    "chainId": 3,
    "chainVersion": 1,
    "targets": [
      { "targetId": 3, "chainId": 3 }
    ]
  },
  {
    "chainId": 4,
    "chainVersion": 1,
    "targets": [
      { "targetId": 4, "chainId": 4 }
    ]
  }
]
EOF
```

---

#### 方案对比

| | 方案 A (单链双副本) | 方案 B (双链双副本) | 方案 C (单副本) |
|---|---|---|---|
| 副本数 | 2 | 2 | 1 |
| 容错 | 一台宕机 OK | 一台宕机 OK | 一台宕机全损 |
| 写 Head | 固定 A | A/B 轮换 | 各自独立 |
| 写转发 | 1 跳 (A→B) | 1 跳 | 0 跳 |
| NDS 适用 | 读走本地，写 Head/Forward 各一次 | Head 负载均衡 | 延迟最低，基准参考 |
| 推荐场景 | 一般测试 | **生产推荐** | 延迟基准 |

---

创建好 Target 和 Chain 后，上传到集群：

```bash
admin_cli --config /opt/3fs/config/hf3fs_fuse_main.toml \
    "create-chain-table --path /tmp/chain_table.json"
```

---

## 第五步：配置 FUSE 客户端（两台机器）

### 5.1 修改 `/opt/3fs/config/hf3fs_fuse_main.toml`

```toml
allow_dev_version = true
cluster_id = 'mycluster'
mountpoint = '/mnt/3fs'

# 客户端出站连接走 TCP
[client]
force_use_tcp = true

# 允许无 RDMA 设备
[ib_devices]
allow_no_usable_devices = true

# MGMTD 客户端配置（注意 section 名称为 [mgmtd]）
[mgmtd]
mgmtd_server_addresses = ['TCP://192.168.1.10:8000']
auto_heartbeat = false

# Storage 客户端网络
[storage.net_client]
force_use_tcp = true

[storage.net_client_for_updates]
force_use_tcp = true
```

### 5.2 设置挂载 Token

```bash
echo -n "admin_token" | sudo tee /opt/3fs/config/fuse_token
sudo chmod 600 /opt/3fs/config/fuse_token
```

### 5.3 挂载

FUSE 客户端不需要 `--app_config` 参数（`hf3fs_fuse_main_app.toml` 为空），只需要 `--config`：

```bash
# 机器 A
sudo /opt/3fs/bin/hf3fs_fuse_main \
    --config /opt/3fs/config/hf3fs_fuse_main.toml

# 机器 B
sudo /opt/3fs/bin/hf3fs_fuse_main \
    --config /opt/3fs/config/hf3fs_fuse_main.toml
```

### 5.4 挂载验证

```bash
# 检查挂载
df -h /mnt/3fs
ls /mnt/3fs

# 测试读写
echo "hello 3fs" | sudo tee /mnt/3fs/test.txt
sudo cat /mnt/3fs/test.txt

# 在机器 A 上写文件
echo "from node A" | sudo tee /mnt/3fs/a.txt

# 在机器 B 上读文件（验证跨节点数据可见）
sudo cat /mnt/3fs/a.txt
```

---

## 常见问题

### Q: MGMTD 启动报错 "IB device not found"

确保以下配置正确：
- `mgmtd_main.toml`: `[ib_devices]` → `allow_no_usable_devices = true`
- `mgmtd_main_app.toml`: `[[server.base.groups]]` 中每个服务组的 `network_type = "TCP"`
- `mgmtd_main_app.toml`: `[server.client]` → `force_use_tcp = true`

### Q: 节点注册不上

检查防火墙允许 8000-9001 端口，确认 `mgmtd_server_addresses` 中地址格式为 `TCP://IP:PORT`。

### Q: FUSE 挂载失败

- 确认 MGMTD、Meta、Storage 都已成功启动并在 `list-nodes` 中可见
- 确认 token 文件内容正确
- FUSE 客户端使用 `--config` 参数指向 `hf3fs_fuse_main.toml`，不使用 `--app_config`

### Q: 配置文件修改后启动报错

三个配置文件各有分工，容易混淆：

| 如果你要修改... | 对应的文件 |
|-----------------|-----------|
| 节点 ID | `*_main_launcher.toml`（所有组件都有） |
| TCP/RDMA 模式、监听端口 | `*_main_app.toml` 的 `[[server.base.groups]]` |
| 客户端出站连接模式 | `*_main_app.toml` 的 `[server.client]` |
| IB 设备设置 | `*_main.toml` 的 `[ib_devices]` |
| MGMTD 地址 | `*_main.toml` 的 `[mgmtd]` 或 `[mgmtd_client]` |
| FDB 连接 | `mgmtd_main.toml` 的 `[fdb]` / `[kv_engine.fdb]` |
| 数据目录 | `storage_main_app.toml` 的 `[server.targets]` |

### Q: 如何添加 NDS 直通 I/O 支持

若机器配备 NDS 硬件，在 Storage 的 `storage_main_app.toml` 中无需额外配置。NDS 直通通过 `NPU_DIRECT_IO` feature flag 自动启用，数据面由 NDS 硬件完成，不影响 TCP 控制面。
