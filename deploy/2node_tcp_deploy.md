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
- 集群元数据存储：可选 **memkv**（内存 KV，测试用，无需外部依赖）或 **FoundationDB**（持久化，生产环境）

---

## 配置文件说明

每个组件使用三个配置文件：

| 文件 | 作用 | 示例核心内容 |
|------|------|------------|
| `*_main.toml` | **核心配置** | 日志、监听端口、服务组、业务逻辑、`[server.client]`、`[server.forward_client]` |
| `*_main_launcher.toml` | **启动器配置** | `allow_dev_version`、`cluster_id`、`[client]`、`[ib_devices]`、`[mgmtd_client]` |
| `*_main_app.toml` | **应用标识** | `allow_empty_node_id`、`node_id`（仅此两个变量，FUSE 的甚至为空） |

启动命令格式：
```bash
./binary --launcher_config *_main_launcher.toml --app_config *_main_app.toml --config *_main.toml
```

---

## 前置准备

### 1. 编译项目

两台机器都执行：

```bash
git clone <repo_url> 3FS
cd 3FS
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo
make -j$(nproc)
```

编译产物位于 `build/bin/`，包括：`mgmtd_main`、`meta_main`、`storage_main`、`hf3fs_fuse_main`、`admin_cli`。

### 2. 选择 KV 存储方案（仅机器 A）

集群元数据需要 KV 存储引擎，二选一：

| 方案 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **memkv** | 测试/开发 | 零依赖，免安装 | 数据掉电丢失 |
| **FoundationDB** | 生产/正式环境 | 持久化存储 | 需要安装 FDB 服务 |

#### 方案：memkv（测试推荐）

无需安装任何东西，后面在 `meta_main.toml` 中设置 `use_memkv = true` 即可。

#### 方案：FoundationDB（生产推荐）

```bash
wget https://github.com/apple/foundationdb/releases/download/7.1.64/foundationdb-clients_7.1.64-1_amd64.deb
wget https://github.com/apple/foundationdb/releases/download/7.1.64/foundationdb-server_7.1.64-1_amd64.deb
sudo dpkg -i foundationdb-clients_7.1.64-1_amd64.deb foundationdb-server_7.1.64-1_amd64.deb
fdbcli --exec "status minimal"
fdbcli --exec "configure new single memory"
```

### 3. 创建目录结构

```bash
sudo mkdir -p /opt/3fs/{bin,config,data}
sudo mkdir -p /var/log/3fs
sudo mkdir -p /mnt/3fs
sudo cp build/bin/* /opt/3fs/bin/
sudo cp configs/*.toml /opt/3fs/config/
```

---

## 第一步：配置 MGMTD（机器 A）

MGMTD 的三个配置文件：

| 参数 | 文件 | 需要修改 |
|------|------|---------|
| `--launcher_config` | `mgmtd_main_launcher.toml` | 添加 `[client] force_use_tcp`、设置 `cluster_id` |
| `--app_config` | `mgmtd_main_app.toml` | 设置 `node_id = 1` |
| `--config` | `mgmtd_main.toml` | 服务组改为 TCP |

### 1.1 修改 `/opt/3fs/config/mgmtd_main_launcher.toml`

```toml
allow_dev_version = true
cluster_id = 'mycluster'

[client]
force_use_tcp = true
```

### 1.2 修改 `/opt/3fs/config/mgmtd_main_app.toml`

```toml
allow_empty_node_id = false
node_id = 1
```

### 1.3 修改 `/opt/3fs/config/mgmtd_main.toml`

将 `[[server.base.groups]]` 中服务组的 `network_type` 从 RDMA 改为 TCP：

```toml
# 第一个服务组：Mgmtd 业务 → TCP
[[server.base.groups]]
services = ['Mgmtd']
network_type = "TCP"
[server.base.groups.listener]
listen_port = 8000
# 其余子配置（io_worker, ibsocket, transport_pool, processor）保持不变

# 第二个服务组：Core → TCP
[[server.base.groups]]
services = ['Core']
network_type = "TCP"
[server.base.groups.listener]
listen_port = 9000
# 其余子配置保持不变
```

### 1.4 启动

```bash
sudo /opt/3fs/bin/mgmtd_main \
    --launcher_config /opt/3fs/config/mgmtd_main_launcher.toml \
    --app_config /opt/3fs/config/mgmtd_main_app.toml \
    --config /opt/3fs/config/mgmtd_main.toml
```

---

## 第二步：配置 Meta（机器 A）

### 2.1 修改 `/opt/3fs/config/meta_main_launcher.toml`

```toml
allow_dev_version = true
cluster_id = 'mycluster'

[client]
force_use_tcp = true

[ib_devices]
allow_no_usable_devices = true

[mgmtd_client]
mgmtd_server_addresses = ['TCP://192.168.1.10:8000']
enable_auto_heartbeat = true
```

### 2.2 修改 `/opt/3fs/config/meta_main_app.toml`

```toml
allow_empty_node_id = false
node_id = 100
```

### 2.3 修改 `/opt/3fs/config/meta_main.toml`

服务组改为 TCP（与 MGMTD 类似）：

```toml
# 第一个服务组：MetaSerde → TCP
[[server.base.groups]]
services = ['MetaSerde']
network_type = "TCP"
[server.base.groups.listener]
listen_port = 8001

# 第二个服务组：Core → TCP
[[server.base.groups]]
services = ['Core']
network_type = "TCP"
[server.base.groups.listener]
listen_port = 9001
```

出站连接走 TCP：

```toml
[server.background_client]
force_use_tcp = true

[server.storage_client.net_client]
force_use_tcp = true

[server.storage_client.net_client_for_updates]
force_use_tcp = true
```

根据所选 KV 方案配置存储引擎：

**memkv 模式**：

```toml
[server.kv_engine]
use_memkv = true
```

**FoundationDB 模式**（需先安装 FDB）：

```toml
[server.fdb]
clusterFile = '/etc/foundationdb/fdb.cluster'

[server.kv_engine]
use_memkv = false
[server.kv_engine.fdb]
clusterFile = '/etc/foundationdb/fdb.cluster'
```

### 2.4 启动

```bash
sudo /opt/3fs/bin/meta_main \
    --launcher_config /opt/3fs/config/meta_main_launcher.toml \
    --app_config /opt/3fs/config/meta_main_app.toml \
    --config /opt/3fs/config/meta_main.toml
```

---

## 第三步：配置 Storage Server（两台机器）

### 3.1 机器 A - Storage (NodeId=10000)

**修改 `/opt/3fs/config/storage_main_launcher.toml`**：

```toml
allow_dev_version = true
cluster_id = 'mycluster'

[client]
force_use_tcp = true

[ib_devices]
allow_no_usable_devices = true

[mgmtd_client]
mgmtd_server_addresses = ['TCP://192.168.1.10:8000']
enable_auto_heartbeat = true
```

**修改 `/opt/3fs/config/storage_main_app.toml`**：

```toml
allow_empty_node_id = false
node_id = 10000
```

**修改 `/opt/3fs/config/storage_main.toml`**：

服务组改为 TCP：

```toml
[[server.base.groups]]
services = ['StorageSerde']
network_type = "TCP"
[server.base.groups.listener]
listen_port = 8000

[[server.base.groups]]
services = ['Core']
network_type = "TCP"
[server.base.groups.listener]
listen_port = 9000
```

出站和转发连接走 TCP：

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
- `storage_main_app.toml`: `node_id = 10001`
- 创建数据目录：`sudo mkdir -p /opt/3fs/data/storage`

### 3.3 启动

```bash
sudo mkdir -p /opt/3fs/data/storage
sudo /opt/3fs/bin/storage_main \
    --launcher_config /opt/3fs/config/storage_main_launcher.toml \
    --app_config /opt/3fs/config/storage_main_app.toml \
    --config /opt/3fs/config/storage_main.toml
```

---

## 第四步：创建 Target 和 Chain

在机器 A 上使用 `admin_cli` 初始化集群。

### 4.1 初始化集群

```bash
# 初始化
admin_cli --config /opt/3fs/config/hf3fs_fuse_main.toml \
    "init-cluster --mgmtd 192.168.1.10:8000 --desc '2-node TCP test cluster'"

# 创建管理员用户
admin_cli --config /opt/3fs/config/hf3fs_fuse_main.toml \
    "create-user --root-admin --token admin_token"

# 列出节点
admin_cli --config /opt/3fs/config/hf3fs_fuse_main.toml \
    "list-nodes"
```

### 4.2 创建 Storage Target

```bash
# 为机器 A 创建 target
admin_cli --config /opt/3fs/config/hf3fs_fuse_main.toml \
    "create-target --node-id 10000 --target-id 1 --disk-index 0 --chain-id 1"

admin_cli --config /opt/3fs/config/hf3fs_fuse_main.toml \
    "create-target --node-id 10000 --target-id 3 --disk-index 0 --chain-id 2"

# 为机器 B 创建 target
admin_cli --config /opt/3fs/config/hf3fs_fuse_main.toml \
    "create-target --node-id 10001 --target-id 2 --disk-index 0 --chain-id 1"

admin_cli --config /opt/3fs/config/hf3fs_fuse_main.toml \
    "create-target --node-id 10001 --target-id 4 --disk-index 0 --chain-id 2"
```

### 4.3 选择 Chain 方案

#### 方案 A：双副本单链
```bash
cat > /tmp/chain_table.json << 'EOF'
[{"chainId":1,"chainVersion":1,"targets":[{"targetId":1,"chainId":1},{"targetId":2,"chainId":1}]}]
EOF
```

#### 方案 B：双副本双链，Head 互换（推荐）
```bash
cat > /tmp/chain_table.json << 'EOF'
[
  {"chainId":1,"chainVersion":1,"targets":[{"targetId":1,"chainId":1},{"targetId":2,"chainId":1}]},
  {"chainId":2,"chainVersion":1,"targets":[{"targetId":2,"chainId":2},{"targetId":1,"chainId":2}]}
]
EOF
```

#### 方案 C：单副本双链（基准测试）
```bash
cat > /tmp/chain_table.json << 'EOF'
[
  {"chainId":3,"chainVersion":1,"targets":[{"targetId":3,"chainId":3}]},
  {"chainId":4,"chainVersion":1,"targets":[{"targetId":4,"chainId":4}]}
]
EOF
```

上传 Chain 配置：
```bash
admin_cli --config /opt/3fs/config/hf3fs_fuse_main.toml \
    "create-chain-table --path /tmp/chain_table.json"
```

---

## 第五步：配置并挂载 FUSE（两台机器）

### 5.1 修改 `/opt/3fs/config/hf3fs_fuse_main_launcher.toml`

```toml
allow_dev_version = true
cluster_id = 'mycluster'

[client]
force_use_tcp = true

[ib_devices]
allow_no_usable_devices = true

[mgmtd_client]
mgmtd_server_addresses = ['TCP://192.168.1.10:8000']
auto_heartbeat = false
```

### 5.2 修改 `/opt/3fs/config/hf3fs_fuse_main.toml`

```toml
mountpoint = '/mnt/3fs'

[client]
force_use_tcp = true

[storage.net_client]
force_use_tcp = true

[storage.net_client_for_updates]
force_use_tcp = true
```

### 5.3 设置 Token

```bash
echo -n "admin_token" | sudo tee /opt/3fs/config/fuse_token
sudo chmod 600 /opt/3fs/config/fuse_token
```

### 5.4 挂载

```bash
# 机器 A 和 B
sudo /opt/3fs/bin/hf3fs_fuse_main \
    --launcher_config /opt/3fs/config/hf3fs_fuse_main_launcher.toml \
    --app_config /opt/3fs/config/hf3fs_fuse_main_app.toml \
    --config /opt/3fs/config/hf3fs_fuse_main.toml
```

### 5.5 验证

```bash
df -h /mnt/3fs
echo "hello 3fs" | sudo tee /mnt/3fs/test.txt
sudo cat /mnt/3fs/test.txt
```

---

## 常见问题

### Q: 启动报错 "IB device not found"

确保以下所有 `force_use_tcp = true` 已设置：
- `mgmtd_main_launcher.toml` → `[client]`
- `meta_main_launcher.toml` → `[client]`
- `meta_main.toml` → `[server.background_client]`、`[server.storage_client.net_client]`、`[server.storage_client.net_client_for_updates]`
- `storage_main_launcher.toml` → `[client]`
- `storage_main.toml` → `[server.client]`、`[server.forward_client]`
- `hf3fs_fuse_main_launcher.toml` → `[client]`
- `hf3fs_fuse_main.toml` → `[client]`、`[storage.net_client]`、`[storage.net_client_for_updates]`

并且所有 `[[server.base.groups]]` 中的 `network_type` 都改为 `"TCP"`。

### Q: `force_use_tcp = true` 放在哪个文件？

| 连接方向 | 文件 | Section |
|---------|------|---------|
| 服务端监听（入站） | `*_main.toml` | `[[server.base.groups]]` → `network_type = "TCP"` |
| 服务端出站 | `*_main.toml` | `[server.client]` 或 `[server.background_client]` |
| 服务端转发 | `*_main.toml` | `[server.forward_client]` |
| 客户端出站 | `*_launcher.toml` | `[client]` |
| Storage 客户端网络 | `hf3fs_fuse_main.toml` | `[storage.net_client]`、`[storage.net_client_for_updates]` |

### Q: FUSE 挂载失败

- 确认 MGMTD、Meta、Storage 都已成功启动
- 先配置 FUSE 的 `*_launcher.toml` 和 `*_main.toml`，再用 admin_cli
- `hf3fs_fuse_main_app.toml` 为空文件，不需要修改
- 如果使用 FDB 模式，确认 `fdbcli --exec "status"` 正常

### Q: memkv 和 FoundationDB 如何切换

由 `meta_main.toml` 中的 `[server.kv_engine]` section 控制：

- memkv: `use_memkv = true`，无需任何其他配置
- FDB: `use_memkv = false`，同时配置 `[server.fdb]` 的 `clusterFile`
- MGMTD 不需要 FDB 配置，KV 只由 Meta 管理

**注意**：切换 KV 引擎后需要重新初始化集群（`init-cluster`），原有数据会丢失。

### Q: NDS 直通 I/O

若机器配备 NDS 硬件，在 Storage 配置中无需额外设置。NDS 直通通过 `NPU_DIRECT_IO` feature flag 自动启用，不影响 TCP 控制面。
