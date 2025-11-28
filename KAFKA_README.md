# Kafka for Kolla-Ansible

> 完整的 Kafka 生态系统部署方案，专为 Kolla-Ansible 设计

[![Kolla](https://img.shields.io/badge/Kolla--Ansible-2023.2%20%7C%202024.1-blue)](https://docs.openstack.org/kolla-ansible/)
[![Kafka](https://img.shields.io/badge/Kafka-3.x%2B-black)](https://kafka.apache.org/)
[![License](https://img.shields.io/badge/License-Apache%202.0-green)](LICENSE)

## 🚀 快速开始

### 1. 复制文件

```bash
# 确保 Kafka 角色已在正确位置
ls ansible/roles/kafka/
```

### 2. 配置

编辑 `/etc/kolla/globals.yml`：

```yaml
enable_kafka: "yes"
enable_kafka_connect: "yes"
enable_kafka_mm2: "yes"
enable_kafka_ui: "yes"

kafka_kraft_mode: "yes"
kafka_ui_password_override: "your-secure-password"
```

编辑 `ansible/inventory/multinode`：

```ini
[kafka]
kafka01
kafka02
kafka03
```

### 3. 部署

```bash
kolla-ansible -i ansible/inventory/multinode deploy --tags kafka
```

### 4. 访问

- **Kafka UI**: http://your-host:8080 (admin / your-secure-password)
- **Kafka Broker**: your-host:9092
- **Kafka Connect**: your-host:8083
- **MirrorMaker 2**: your-host:8084

## 📚 文档

| 文档 | 说明 |
|------|------|
| [📖 KAFKA_SUMMARY.md](KAFKA_SUMMARY.md) | **从这里开始** - 项目总结 |
| [📘 KAFKA_DEPLOYMENT_GUIDE.md](KAFKA_DEPLOYMENT_GUIDE.md) | 完整部署指南 |
| [🐳 KAFKA_DOCKER_BUILD.md](KAFKA_DOCKER_BUILD.md) | 镜像构建指南 |
| [🔄 KAFKA_MM2_MIGRATION_GUIDE.md](KAFKA_MM2_MIGRATION_GUIDE.md) | MM2 迁移指南 |
| [📋 FILES_CREATED.md](FILES_CREATED.md) | 文件清单 |
| [📖 KAFKA_IMPLEMENTATION_README.md](KAFKA_IMPLEMENTATION_README.md) | 实现细节 |

## ✨ 特性

- ✅ **4 个独立容器**: Broker, Connect, MirrorMaker 2, UI
- ✅ **KRaft 模式**: 无需 Zookeeper
- ✅ **完全集成**: 与 Kolla-Ansible 无缝集成
- ✅ **高可用**: 支持多节点集群
- ✅ **安全**: SASL/SSL 认证支持
- ✅ **跨集群同步**: MirrorMaker 2 支持
- ✅ **Web 管理**: Kafka UI 界面
- ✅ **生产就绪**: 性能优化和监控

## 🏗️ 架构

```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Kafka Broker   │  │  Kafka Broker   │  │  Kafka Broker   │
│    Port: 9092   │  │    Port: 9092   │  │    Port: 9092   │
└─────────────────┘  └─────────────────┘  └─────────────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              │
┌──────────────────────────────┴───────────────────────────────┐
│                                                                │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────┐ │
│  │ Kafka Connect    │  │ MirrorMaker 2    │  │ Kafka UI   │ │
│  │   Port: 8083     │  │   Port: 8084     │  │ Port: 8080 │ │
│  └──────────────────┘  └──────────────────┘  └────────────┘ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 📦 包含组件

| 组件 | 容器名 | 端口 | 说明 |
|------|--------|------|------|
| Kafka Broker | `kafka_broker` | 9092 | 消息队列核心 |
| Kafka Connect | `kafka_connect` | 8083 | 数据集成平台 |
| MirrorMaker 2 | `kafka_mm2` | 8084 | 跨集群同步 |
| Kafka UI | `kafka_ui` | 8080 | Web 管理界面 |

## 🎯 适用场景

- ✅ OpenStack 环境中部署 Kafka
- ✅ 需要统一配置管理
- ✅ 跨数据中心数据同步
- ✅ 容器化部署
- ✅ 快速原型和生产部署

## 🔧 核心配置

### MirrorMaker 2 示例（阿里云 Kafka）

```yaml
# 源集群（k8s 内部）
kafka_mm2_source_cluster: "source"
kafka_mm2_source_bootstrap_servers: "kafka-0:9094,kafka-1:9094,kafka-2:9094"

# 目标集群（阿里云）
kafka_mm2_target_cluster: "destination"
kafka_mm2_target_bootstrap_servers: "alikafka-xxx-1.aliyuncs.com:9093,..."
kafka_mm2_target_security_protocol: "SASL_SSL"
kafka_mm2_target_sasl_mechanism: "PLAIN"
kafka_mm2_target_sasl_jaas_config: 'org.apache.kafka.common.security.plain.PlainLoginModule required username="xxx" password="xxx";'

# 同步配置
kafka_mm2_topics_pattern: "model-token-usage"  # 或 ".*"
kafka_mm2_replication_factor: "3"
```

## 🛠️ 常用命令

```bash
# 部署
kolla-ansible -i inventory/multinode deploy --tags kafka

# 重新配置
kolla-ansible -i inventory/multinode reconfigure --tags kafka

# 停止
kolla-ansible -i inventory/multinode stop --tags kafka

# 查看容器
docker ps | grep kafka

# 查看日志
docker logs -f kafka_broker
docker logs -f kafka_mm2

# 测试 Kafka
docker exec -it kafka_broker bash
kafka-topics.sh --list --bootstrap-server localhost:9092
```

## 📊 验证

```bash
# 1. 检查容器运行
docker ps | grep kafka

# 2. 测试 Broker
kafka-topics.sh --create --topic test --bootstrap-server localhost:9092 --partitions 3 --replication-factor 3

# 3. 测试 Connect
curl http://localhost:8083/

# 4. 测试 MM2
curl http://localhost:8084/connectors

# 5. 访问 UI
open http://localhost:8080
```

## 🐛 故障排查

```bash
# 查看日志
docker logs kafka_broker
docker logs kafka_mm2

# 检查配置
docker exec kafka_broker cat /opt/kafka/config/server.properties

# 测试网络
docker exec kafka_mm2 telnet target-broker 9092

# 检查证书
docker exec kafka_mm2 ls -la /etc/kafka/secrets/
```

## 📝 配置变量参考

最常用的变量：

```yaml
# 启用组件
enable_kafka: "yes"
enable_kafka_connect: "yes"
enable_kafka_mm2: "yes"
enable_kafka_ui: "yes"

# 基础配置
kafka_kraft_mode: "yes"
kafka_default_replication_factor: "3"
kafka_port: "9092"

# JVM 内存
kafka_broker_heap_opts: "-Xms2G -Xmx2G"

# UI 认证
kafka_ui_username: "admin"
kafka_ui_password_override: "changeme"
```

完整变量列表请参考 [KAFKA_DEPLOYMENT_GUIDE.md](KAFKA_DEPLOYMENT_GUIDE.md)。

## 🎓 使用场景

### 场景 1: 单机测试
```bash
kolla-ansible -i inventory/all-in-one deploy --tags kafka
```

### 场景 2: 生产集群（3节点）
```ini
[kafka]
kafka01
kafka02
kafka03
```

### 场景 3: 跨集群同步
配置 MirrorMaker 2，实现数据中心间数据同步。

## 📈 性能建议

```yaml
# 生产环境推荐配置
kafka_broker_heap_opts: "-Xms8G -Xmx8G"
kafka_default_replication_factor: "3"
kafka_min_insync_replicas: "2"

# MirrorMaker 2
kafka_mm2_tasks_max: "12"
kafka_mm2_consumer_fetch_max_bytes: "104857600"
```

## 🔐 安全

1. 修改默认密码
2. 启用 TLS/SSL
3. 配置 SASL 认证
4. 网络隔离
5. 定期更新

详见 [KAFKA_DEPLOYMENT_GUIDE.md](KAFKA_DEPLOYMENT_GUIDE.md) 的安全部分。

## 📋 需求

- Kolla-Ansible: 2023.2 (Zed) 或 2024.1 (Caracal)
- Docker: 20.10+
- Python: 3.8+
- Ansible: 2.14+

## 🗂️ 文件结构

```
ansible/roles/kafka/
├── defaults/main.yml              # 默认配置
├── handlers/main.yml              # 处理器
├── tasks/                         # 任务
├── templates/                     # 配置模板
└── vars/main.yml                  # 变量
```

完整清单: [FILES_CREATED.md](FILES_CREATED.md)

## 🤝 贡献

欢迎：
- 报告问题
- 提交改进
- 分享使用经验
- 贡献文档

## 📄 许可证

Apache License 2.0

## 🔗 资源

- [Apache Kafka](https://kafka.apache.org/)
- [Kolla-Ansible](https://docs.openstack.org/kolla-ansible/)
- [MirrorMaker 2](https://cwiki.apache.org/confluence/display/KAFKA/KIP-382%3A+MirrorMaker+2.0)

## 📞 支持

1. 📖 查阅文档（特别是 KAFKA_SUMMARY.md）
2. 🔍 检查日志和错误信息
3. 💬 提交 Issue
4. 📧 联系维护者

## ⭐ 版本

- **当前版本**: 1.0.0
- **发布日期**: 2024-11
- **Kolla 版本**: 2023.2 / 2024.1
- **Kafka 版本**: 3.x+

---

## 🎉 开始使用

**推荐阅读顺序**:
1. 本文档（快速了解）
2. [KAFKA_SUMMARY.md](KAFKA_SUMMARY.md)（项目总结）
3. [KAFKA_DEPLOYMENT_GUIDE.md](KAFKA_DEPLOYMENT_GUIDE.md)（详细部署）

**立即开始**:
```bash
# 1. 配置
vim /etc/kolla/globals.yml
vim ansible/inventory/multinode

# 2. 部署
kolla-ansible -i ansible/inventory/multinode deploy --tags kafka

# 3. 访问
open http://your-host:8080
```

祝您使用愉快！🚀
