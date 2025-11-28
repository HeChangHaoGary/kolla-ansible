# Kafka for Kolla-Ansible - 完整实现文档

## 项目概述

这是一个为 Kolla-Ansible 开发的完整 Kafka 部署方案，支持在 OpenStack 环境中快速部署和管理 Kafka 生态系统。

### 版本信息
- **Kolla-Ansible 版本**: 2023.2 (Zed) / 2024.1 (Caracal)
- **Kafka 版本**: 3.x+
- **支持的模式**: KRaft (无需 Zookeeper)

## 架构设计

### 多容器部署架构

```
┌──────────────────────────────────────────────────────────────┐
│                    Kafka 生态系统                              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐  │
│  │  Kafka Broker   │  │  Kafka Broker   │  │ Kafka Broker│  │
│  │   (Container)   │  │   (Container)   │  │ (Container) │  │
│  │   Node 1        │  │   Node 2        │  │  Node 3     │  │
│  │   Port: 9092    │  │   Port: 9092    │  │  Port: 9092 │  │
│  └─────────────────┘  └─────────────────┘  └─────────────┘  │
│          │                     │                    │         │
│          └─────────────────────┼────────────────────┘         │
│                                │                              │
│  ┌─────────────────────────────┼───────────────────────────┐ │
│  │         Kafka Connect Cluster (Optional)                 │ │
│  │         Port: 8083                                       │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │         MirrorMaker 2 (Optional)                         │ │
│  │         Cross-cluster replication                        │ │
│  │         Port: 8084                                       │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │         Kafka UI (Optional)                              │ │
│  │         Web Management Interface                         │ │
│  │         Port: 8080                                       │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### 组件说明

| 组件 | 容器名 | 端口 | 用途 |
|-----|--------|------|------|
| Kafka Broker | `kafka_broker` | 9092 | 消息队列核心服务 |
| Kafka Connect | `kafka_connect` | 8083 | 数据集成平台 |
| MirrorMaker 2 | `kafka_mm2` | 8084 | 跨集群数据同步 |
| Kafka UI | `kafka_ui` | 8080 | Web 管理界面 |

## 文件结构

```
workspace/
├── ansible/
│   ├── roles/
│   │   └── kafka/                          # Kafka 角色
│   │       ├── defaults/
│   │       │   └── main.yml                # 默认变量（镜像、端口、配置等）
│   │       ├── handlers/
│   │       │   └── main.yml                # 容器重启处理器
│   │       ├── tasks/
│   │       │   ├── main.yml                # 任务入口
│   │       │   ├── config.yml              # 配置任务
│   │       │   ├── deploy.yml              # 部署任务
│   │       │   ├── check-containers.yml    # 容器检查
│   │       │   ├── deploy-containers.yml   # 容器部署
│   │       │   ├── precheck.yml            # 预检查
│   │       │   ├── pull.yml                # 镜像拉取
│   │       │   ├── reconfigure.yml         # 重新配置
│   │       │   ├── stop.yml                # 停止服务
│   │       │   └── upgrade.yml             # 升级
│   │       ├── templates/
│   │       │   ├── kafka-broker.json.j2           # Broker 容器配置
│   │       │   ├── kafka-connect.json.j2          # Connect 容器配置
│   │       │   ├── kafka-mm2.json.j2              # MM2 容器配置
│   │       │   ├── kafka-ui.json.j2               # UI 容器配置
│   │       │   ├── server.properties.j2           # Kafka Broker 配置
│   │       │   ├── connect-distributed.properties.j2  # Connect 配置
│   │       │   ├── mm2.properties.j2              # MirrorMaker 2 配置
│   │       │   ├── log4j.properties.j2            # 日志配置
│   │       │   └── connect-log4j.properties.j2    # Connect 日志配置
│   │       └── vars/
│   │           └── main.yml                # 角色变量
│   ├── kafka.yml                           # Kafka 独立 playbook
│   ├── site.yml                            # 主 playbook（已集成 Kafka）
│   ├── inventory/
│   │   ├── all-in-one                      # 单节点清单（已更新）
│   │   └── multinode                       # 多节点清单（已更新）
│   └── group_vars/
│       └── kafka.yml                       # Kafka 组变量示例
│
├── KAFKA_DEPLOYMENT_GUIDE.md               # 📘 部署指南
├── KAFKA_DOCKER_BUILD.md                   # 🐳 镜像构建指南
├── KAFKA_MM2_MIGRATION_GUIDE.md            # 🔄 MM2 迁移指南
├── KAFKA_IMPLEMENTATION_README.md          # 📖 本文档
└── mm2.properties                          # 您的原始 MM2 配置（已更新）
```

## 核心特性

### ✅ 已实现的功能

1. **完整的 Kafka 生态系统部署**
   - Kafka Broker (KRaft 模式)
   - Kafka Connect (分布式模式)
   - MirrorMaker 2 (跨集群同步)
   - Kafka UI (Web 管理界面)

2. **Kolla-Ansible 集成**
   - 完整的 Ansible 角色结构
   - 符合 Kolla 容器管理模式
   - 支持所有 Kolla 操作：deploy, reconfigure, upgrade, stop
   - 集成到主 playbook (site.yml)

3. **配置管理**
   - Jinja2 模板化配置
   - 支持自定义配置覆盖
   - 环境变量注入
   - 密钥管理（SSL/SASL）

4. **高可用性**
   - KRaft 模式（无需 Zookeeper）
   - 多节点集群支持
   - 可配置的复制因子
   - 自动故障转移

5. **安全性**
   - SASL/SSL 认证支持
   - 用户密码管理
   - 网络隔离
   - 访问控制

6. **监控和健康检查**
   - Docker 健康检查
   - JMX 监控端口
   - 详细的日志记录
   - REST API 状态查询

7. **数据同步 (MirrorMaker 2)**
   - 单向/双向同步
   - Topic 过滤
   - 自动 offset 同步
   - 跨数据中心复制

## 快速开始

### 1. 准备工作

```bash
# 克隆或更新 Kolla-Ansible
cd /workspace

# 检查 Kafka 角色是否存在
ls -la ansible/roles/kafka/
```

### 2. 配置 Inventory

编辑 `ansible/inventory/multinode`：

```ini
# Kafka 节点
[kafka]
kafka01
kafka02
kafka03
```

### 3. 配置 Globals

编辑 `/etc/kolla/globals.yml`：

```yaml
# 启用 Kafka
enable_kafka: "yes"
enable_kafka_connect: "yes"
enable_kafka_mm2: "yes"
enable_kafka_ui: "yes"

# Kafka 配置
kafka_kraft_mode: "yes"
kafka_default_replication_factor: "3"

# Kafka UI
kafka_ui_username: "admin"
kafka_ui_password_override: "secure-password"
```

### 4. 部署

```bash
# 预检查
kolla-ansible -i ansible/inventory/multinode prechecks --tags kafka

# 部署
kolla-ansible -i ansible/inventory/multinode deploy --tags kafka
```

### 5. 验证

```bash
# 检查容器
docker ps | grep kafka

# 访问 Kafka UI
http://<your-host>:8080
```

## 详细文档

### 📘 部署指南
[KAFKA_DEPLOYMENT_GUIDE.md](KAFKA_DEPLOYMENT_GUIDE.md) - 完整的部署步骤和配置说明

### 🐳 镜像构建指南
[KAFKA_DOCKER_BUILD.md](KAFKA_DOCKER_BUILD.md) - Docker 镜像构建和自定义

### 🔄 MM2 迁移指南
[KAFKA_MM2_MIGRATION_GUIDE.md](KAFKA_MM2_MIGRATION_GUIDE.md) - 将现有 MM2 迁移到 Kolla

## 配置变量参考

### 关键变量

#### Kafka Broker
```yaml
kafka_port: "9092"                          # Kafka 端口
kafka_kraft_mode: "yes"                     # KRaft 模式
kafka_broker_heap_opts: "-Xms2G -Xmx2G"    # JVM 堆内存
kafka_default_replication_factor: "3"       # 复制因子
kafka_log_retention_hours: "168"            # 日志保留时间
```

#### Kafka Connect
```yaml
kafka_connect_port: "8083"                  # Connect REST API 端口
kafka_connect_heap_opts: "-Xms1G -Xmx1G"   # JVM 堆内存
kafka_connect_group_id: "connect-cluster"   # 集群 ID
```

#### MirrorMaker 2
```yaml
kafka_mm2_source_cluster: "source"                    # 源集群名称
kafka_mm2_target_cluster: "destination"               # 目标集群名称
kafka_mm2_source_bootstrap_servers: "host1:9092,..."  # 源集群地址
kafka_mm2_target_bootstrap_servers: "host2:9092,..."  # 目标集群地址
kafka_mm2_topics_pattern: ".*"                        # Topic 过滤
kafka_mm2_replication_factor: "3"                     # 复制因子
```

#### Kafka UI
```yaml
kafka_ui_port: "8080"                       # Web UI 端口
kafka_ui_username: "admin"                  # 登录用户名
kafka_ui_password_override: "changeme"      # 登录密码
```

## 运维操作

### 部署新集群
```bash
kolla-ansible -i inventory deploy --tags kafka
```

### 重新配置
```bash
kolla-ansible -i inventory reconfigure --tags kafka
```

### 升级
```bash
kolla-ansible -i inventory upgrade --tags kafka
```

### 停止服务
```bash
kolla-ansible -i inventory stop --tags kafka
```

### 单独操作某个组件
```bash
# 仅部署 Broker
kolla-ansible -i inventory deploy --tags kafka-broker

# 仅部署 Connect
kolla-ansible -i inventory deploy --tags kafka-connect

# 仅部署 MirrorMaker 2
kolla-ansible -i inventory deploy --tags kafka-mm2
```

## 监控和日志

### 日志位置
```bash
# Broker 日志
/var/log/kolla/kafka/

# Connect 日志
/var/log/kolla/kafka-connect/

# MirrorMaker 2 日志
/var/log/kolla/kafka-mm2/

# Kafka UI 日志
/var/log/kolla/kafka-ui/
```

### 查看日志
```bash
# 实时查看 Broker 日志
docker logs -f kafka_broker

# 查看 MM2 日志
docker logs -f kafka_mm2
```

### JMX 监控
```bash
# Broker JMX 端口: 9999
# 可以使用 JConsole 或 Prometheus JMX Exporter 连接
```

## 故障排查

### 常见问题

#### 1. 容器无法启动
```bash
# 查看容器日志
docker logs kafka_broker

# 检查配置文件
docker exec kafka_broker cat /opt/kafka/config/server.properties

# 检查数据目录权限
docker exec kafka_broker ls -la /var/lib/kafka/data/
```

#### 2. MirrorMaker 2 无法连接
```bash
# 测试网络连通性
docker exec kafka_mm2 telnet source-broker 9092

# 检查 SSL 证书
docker exec kafka_mm2 ls -la /etc/kafka/secrets/

# 查看连接器状态
curl http://localhost:8084/connectors/MirrorSourceConnector/status
```

#### 3. Kafka UI 无法访问
```bash
# 检查端口
netstat -tlnp | grep 8080

# 检查防火墙
firewall-cmd --list-ports

# 查看容器状态
docker ps | grep kafka_ui
```

## 性能调优

### Broker 调优
```yaml
# JVM 堆内存（根据服务器内存调整）
kafka_broker_heap_opts: "-Xms8G -Xmx8G -XX:+UseG1GC"

# 网络和 I/O 线程
kafka_custom_config: |
  num.network.threads=8
  num.io.threads=16
  
# 日志段大小
kafka_log_segment_bytes: "1073741824"  # 1GB
```

### Connect 调优
```yaml
kafka_connect_heap_opts: "-Xms4G -Xmx4G"
kafka_connect_tasks_max: "8"
```

### MirrorMaker 2 调优
```yaml
kafka_mm2_tasks_max: "12"
kafka_mm2_consumer_fetch_max_bytes: "104857600"  # 100MB
kafka_mm2_producer_batch_size: "65536"           # 64KB
```

## 安全最佳实践

1. **更改默认密码**
   ```yaml
   kafka_ui_password_override: "strong-random-password"
   ```

2. **启用 TLS**
   ```yaml
   kafka_enable_ssl: "yes"
   kafka_ssl_keystore_location: "/path/to/keystore"
   ```

3. **配置 SASL 认证**
   ```yaml
   kafka_enable_sasl: "yes"
   kafka_sasl_mechanism: "SCRAM-SHA-512"
   ```

4. **网络隔离**
   - 使用防火墙限制访问
   - 配置安全组规则
   - 使用内部网络

5. **定期备份**
   - 备份配置文件
   - 备份 Kafka 数据
   - 测试恢复流程

## 扩展和定制

### 添加自定义配置
```yaml
# 在 globals.yml 中
kafka_custom_config: |
  # 自定义的 Kafka 配置
  compression.type=lz4
  log.cleanup.policy=compact
```

### 添加 Connect 插件
```yaml
kafka_connect_extra_volumes:
  - "/opt/kafka-plugins:/opt/kafka/plugins:ro"
```

### 修改镜像
```yaml
# 使用自定义镜像
kafka_broker_image: "your-registry/kafka"
kafka_broker_tag: "3.6.0-custom"
```

## 贡献和支持

### 报告问题
如果遇到问题，请提供：
- Kolla-Ansible 版本
- Kafka 版本
- 错误日志
- 配置文件

### 功能请求
欢迎提交功能请求和改进建议。

## 许可证

Apache License 2.0

## 相关资源

- [Apache Kafka 官方文档](https://kafka.apache.org/documentation/)
- [Kolla-Ansible 官方文档](https://docs.openstack.org/kolla-ansible/)
- [Kafka Connect 文档](https://kafka.apache.org/documentation/#connect)
- [MirrorMaker 2 文档](https://cwiki.apache.org/confluence/display/KAFKA/KIP-382%3A+MirrorMaker+2.0)

## 更新日志

### v1.0.0 (2024-11)
- ✅ 初始版本
- ✅ 支持 Kafka Broker (KRaft 模式)
- ✅ 支持 Kafka Connect
- ✅ 支持 MirrorMaker 2
- ✅ 支持 Kafka UI
- ✅ 完整的 Kolla-Ansible 集成
- ✅ 多容器架构
- ✅ SASL/SSL 支持
- ✅ 完整的文档

## 下一步计划

- [ ] 添加 Schema Registry 支持
- [ ] 集成 Prometheus 监控
- [ ] 添加 KSQL 支持
- [ ] 自动化测试套件
- [ ] CI/CD 集成示例
- [ ] 性能基准测试
- [ ] 多数据中心部署指南

---

**祝您使用愉快！**
