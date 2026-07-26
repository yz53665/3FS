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
- 数据存储使用 **memkv**（内存 KV），无需 FoundationDB

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

### 2. 创建目录结构

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

### 3. 确保网络互通

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

### 1.1 修改 `/opt/3fs/config/mgmtd_main_launcher.toml`

```toml
allow_dev_version = true
cluster_id = 'mycluster'
use_memkv = true                    # 使用内存 KV，不依赖 FDB

[mgmtd_client]
mgmtd_server_addresses = ['TCP://192.168.1.10:8000']
```

### 1.2 修改 `/opt/3fs/config/mgmtd_main_app.toml`

```toml
allow_empty_node_id = false
node_id = 1                         # MGMTD 固定使用 NodeId=1
```

### 1.3 修改 `/opt/3fs/config/mgmtd_main.toml`

在 `[server]` 段下添加 TCP 配置，并允许无 IB 设备：

```toml
# 关键：允许无 IB 设备
[ib_devices]
allow_no_usable_devices = true

# 服务端监听改为 TCP
[server.groups.0]
network_type = "TCP"
services = ["Mgmtd"]

[server.groups.1]
network_type = "TCP"
services = ["Core"]
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

### 2.1 修改 `/opt/3fs/config/meta_main_launcher.toml`

```toml
allow_dev_version = true
cluster_id = 'mycluster'

[mgmtd_client]
mgmtd_server_addresses = ['TCP://192.168.1.10:8000']
enable_auto_heartbeat = true

[client]
force_use_tcp = true                # TCP 模式

[ib_devices]
allow_no_usable_devices = true
```

### 2.2 修改 `/opt/3fs/config/meta_main_app.toml`

```toml
allow_empty_node_id = false
node_id = 100
```

### 2.3 修改 `/opt/3fs/config/meta_main.toml`

在 `[server]` 段添加 TCP 配置。

### 2.4 启动 Meta

```bash
sudo /opt/3fs/bin/meta_main \
    --launcher_config /opt/3fs/config/meta_main_launcher.toml \
    --app_config /opt/3fs/config/meta_main_app.toml \
    --config /opt/3fs/config/meta_main.toml
```

---

## 第三步：配置 Storage（两台机器）

### 3.1 机器 A - Storage (NodeId=10000)

修改 `/opt/3fs/config/storage_main_launcher.toml`：

```toml
allow_dev_version = true
cluster_id = 'mycluster'

[mgmtd_client]
mgmtd_server_addresses = ['TCP://192.168.1.10:8000']
enable_auto_heartbeat = true

[client]
force_use_tcp = true                # TCP 模式

[ib_devices]
allow_no_usable_devices = true
```

修改 `/opt/3fs/config/storage_main_app.toml`：

```toml
allow_empty_node_id = false
node_id = 10000
```

修改 `/opt/3fs/config/storage_main.toml` 的服务端配置（关键：TCP 监听 + 数据目录）：

```toml
[ib_devices]
allow_no_usable_devices = true

[server.groups.0]
network_type = "TCP"
services = ["StorageSerde"]
listener.listen_port = 8000

[server.groups.1]
network_type = "TCP"
services = ["Core"]
listener.listen_port = 9000

# 数据存储配置
[storage]
target_paths = ['/opt/3fs/data/storage']

# client 和 forward_client 都需要 force_use_tcp
[server.client]
force_use_tcp = true

[server.forward_client]
force_use_tcp = true
```

启动：

```bash
sudo mkdir -p /opt/3fs/data/storage
sudo /opt/3fs/bin/storage_main \
    --launcher_config /opt/3fs/config/storage_main_launcher.toml \
    --app_config /opt/3fs/config/storage_main_app.toml \
    --config /opt/3fs/config/storage_main.toml
```

### 3.2 机器 B - Storage (NodeId=10001)

操作与机器 A 相同，区别在于：

- `storage_main_app.toml`: `node_id = 10001`
- 启动前创建数据目录: `sudo mkdir -p /opt/3fs/data/storage`

---

## 第四步：创建 Target 和 Chain

在机器 A 上使用 `admin_cli` 初始化集群。

### 4.1 创建 admin_cli 配置文件

创建 `/opt/3fs/config/admin_cli.toml`：

```toml
cluster_id = 'mycluster'

[client]
force_use_tcp = true

[ib_devices]
allow_no_usable_devices = true

[mgmtd_client]
mgmtd_server_addresses = ['TCP://192.168.1.10:8000']
```

### 4.2 初始化集群

```bash
# 1. 初始化
admin_cli --config /opt/3fs/config/admin_cli.toml \
    "init-cluster --mgmtd 192.168.1.10:8000 --desc '2-node TCP test cluster'"

# 2. 创建管理员用户
admin_cli --config /opt/3fs/config/admin_cli.toml \
    "create-user --root-admin --token admin_token"

# 3. 列出节点，确认 MGMTD/Meta/Storage 都已上线
admin_cli --config /opt/3fs/config/admin_cli.toml \
    "list-nodes"
```

### 4.3 创建 Storage Target

```bash
# 为机器 A 的 Storage 创建 target (target_id 和 node_id 对应)
admin_cli --config /opt/3fs/config/admin_cli.toml \
    "create-target --node-id 10000 --target-id 1 --disk-index 0 --chain-id 1"

# 为机器 B 的 Storage 创建 target
admin_cli --config /opt/3fs/config/admin_cli.toml \
    "create-target --node-id 10001 --target-id 2 --disk-index 0 --chain-id 1"
```

### 4.4 创建 Chain Table

```toml
# 写入链配置 JSON
cat > /tmp/chain_table.json << 'EOF'
{
  "chainId": 1,
  "chainVersion": 1,
  "targets": [
    { "targetId": 1, "chainId": 1 },
    { "targetId": 2, "chainId": 1 }
  ]
}
EOF

admin_cli --config /opt/3fs/config/admin_cli.toml \
    "create-chain-table --path /tmp/chain_table.json"
```

---

## 第五步：挂载 FUSE（两台机器）

### 5.1 创建 FUSE 配置文件

创建 `/opt/3fs/config/hf3fs_fuse.toml`：

```toml
# launcher 配置
allow_dev_version = true
cluster_id = 'mycluster'
mountpoint = '/mnt/3fs'
token_file = '/opt/3fs/config/fuse_token'

[mgmtd_client]
mgmtd_server_addresses = ['TCP://192.168.1.10:8000']
enable_auto_heartbeat = false        # FUSE 客户端不需要心跳

[client]
force_use_tcp = true

[ib_devices]
allow_no_usable_devices = true

# Storage 客户端连接
[storage.net_client]
force_use_tcp = true

[storage.net_client_for_updates]
force_use_tcp = true
```

### 5.2 创建 Token 文件

```bash
echo -n "admin_token" | sudo tee /opt/3fs/config/fuse_token
sudo chmod 600 /opt/3fs/config/fuse_token
```

### 5.3 挂载

```bash
# 机器 A
sudo /opt/3fs/bin/hf3fs_fuse_main \
    --launcher_config /opt/3fs/config/hf3fs_fuse.toml \
    --config /opt/3fs/config/hf3fs_fuse_main.toml

# 机器 B
sudo /opt/3fs/bin/hf3fs_fuse_main \
    --launcher_config /opt/3fs/config/hf3fs_fuse.toml \
    --config /opt/3fs/config/hf3fs_fuse_main.toml
```

### 5.4 验证

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

## 配置清单速查表

### TCP-Only 必须修改的配置项

| 配置位置 | 配置项 | 值 | 说明 |
|---|---|---|---|
| 所有 `*_launcher.toml` | `cluster_id` | `'mycluster'` | 集群 ID，所有节点一致 |
| 所有 `*_launcher.toml` | `mgmtd_client.mgmtd_server_addresses` | `['TCP://192.168.1.10:8000']` | MGMTD 地址 |
| 所有 `*_launcher.toml` | `ib_devices.allow_no_usable_devices` | `true` | 允许无 IB 设备 |
| 所有 `*_app.toml` | `node_id` | 各节点唯一值 | 节点标识 |
| MGMTD launcher | `use_memkv` | `true` | 内存 KV，免 FDB |
| 服务端 `*_main.toml` | `server.groups.0.network_type` | `"TCP"` | 业务服务走 TCP |
| 服务端 `*_main.toml` | `server.client.force_use_tcp` | `true` | 出站连接走 TCP |
| 客户端 `*_main.toml` | `client.force_use_tcp` | `true` | 出站连接走 TCP |
| Storage | `server.forward_client.force_use_tcp` | `true` | Forwarding 走 TCP |

### NodeId 分配

| 机器 | 组件 | NodeId |
|---|---|---|
| A | MGMTD | 1 |
| A | Meta | 100 |
| A | Storage | 10000 |
| B | Storage | 10001 |

### 端口分配

| 组件 | 业务端口 | Core 端口 |
|---|---|---|
| MGMTD | TCP:8000 | TCP:9000 |
| Meta | TCP:8001 | TCP:9001 |
| Storage A | TCP:8000 | TCP:9000 |
| Storage B | TCP:8000 | TCP:9000 |

---

## 常见问题

### Q: MGMTD 启动报错 "IB device not found"

确保 `ib_devices.allow_no_usable_devices = true`，且 `server.groups.0.network_type = "TCP"`。

### Q: 节点注册不上

检查防火墙允许 8000-9001 端口，确认 `mgmtd_server_addresses` 中地址格式为 `TCP://IP:PORT`。

### Q: FUSE 挂载失败

检查 token 文件是否正确，确认 meta/storage 节点都已成功注册。

### Q: 如何添加 NDS 直通 I/O 支持

若机器配备 NDS 硬件，在 Storage 的 `storage_main.toml` 中无需额外配置。NDS 直通通过 `NPU_DIRECT_IO` feature flag 自动启用，数据面由 NDS 硬件完成，不影响 TCP 控制面。
