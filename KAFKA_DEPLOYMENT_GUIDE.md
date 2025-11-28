# Kafka Deployment Guide for Kolla-Ansible

## 概述

这是一个完整的 Kolla-Ansible Kafka 部署方案，支持以下组件：
- **Kafka Broker** - Kafka 消息代理服务（支持 KRaft 模式）
- **Kafka Connect** - 数据集成平台
- **MirrorMaker 2** - 跨集群数据同步
- **Kafka UI** - Web 管理界面

## 架构设计

### 多容器部署方案（推荐）

每个 Kafka 组件运行在独立容器中：

```
┌─────────────────┐
│  Kafka Broker   │  Port: 9092
│   (Container)   │  JMX: 9999
└─────────────────┘

┌─────────────────┐
│ Kafka Connect   │  Port: 8083
│   (Container)   │  REST API
└─────────────────┘

┌─────────────────┐
│ MirrorMaker 2   │  Port: 8084
│   (Container)   │  REST API
└─────────────────┘

┌─────────────────┐
│   Kafka UI      │  Port: 8080
│   (Container)   │  Web Interface
└─────────────────┘
```

### 优势
- ✅ 职责分离，易于管理
- ✅ 独立扩展和升级
- ✅ 故障隔离
- ✅ 符合容器化最佳实践

## 部署步骤

### 1. 配置 Inventory

#### All-in-One 部署
```ini
# ansible/inventory/all-in-one
[control]
localhost       ansible_connection=local

# Kafka 组已自动添加到文件末尾
[kafka:children]
control
```

#### 多节点部署（生产环境推荐）
```ini
# ansible/inventory/multinode

# 推荐：使用至少 3 个节点部署 Kafka Broker
[kafka]
kafka01
kafka02
kafka03

# 或者复用现有的 control 节点
[kafka:children]
control

# Kafka 子组
[kafka-broker:children]
kafka

[kafka-connect:children]
kafka

[kafka-mm2:children]
kafka

[kafka-ui:children]
kafka
```

### 2. 配置 globals.yml

在 `/etc/kolla/globals.yml` 中添加：

```yaml
####################
# Kafka Configuration
####################
enable_kafka: "yes"
enable_kafka_connect: "yes"
enable_kafka_mm2: "yes"
enable_kafka_ui: "yes"

# Kafka 集群 ID
kafka_cluster_id_override: "kafka-kolla-cluster-1"

# 使用 KRaft 模式（无需 Zookeeper）
kafka_kraft_mode: "yes"

# 复制因子（建议 3）
kafka_default_replication_factor: "3"
kafka_min_insync_replicas: "2"

# JVM 内存设置
kafka_broker_heap_opts: "-Xms4G -Xmx4G"
kafka_connect_heap_opts: "-Xms2G -Xmx2G"
kafka_mm2_heap_opts: "-Xms2G -Xmx2G"

# Kafka UI 配置
kafka_ui_username: "admin"
kafka_ui_password_override: "your-secure-password"
```

### 3. MirrorMaker 2 配置（可选）

如果需要跨集群数据同步，配置 MirrorMaker 2：

```yaml
# MirrorMaker 2 源集群配置
kafka_mm2_source_cluster: "source"
kafka_mm2_source_bootstrap_servers: "10.220.244.176:9092,10.220.244.177:9092,10.220.244.178:9092"
kafka_mm2_source_security_protocol: "PLAINTEXT"

# MirrorMaker 2 目标集群配置
kafka_mm2_target_cluster: "destination"
kafka_mm2_target_bootstrap_servers: "10.220.57.24:9092,10.220.57.25:9092,10.220.57.26:9092"

# 同步配置
kafka_mm2_topics_pattern: ".*"          # 同步所有 topic
kafka_mm2_topics_exclude: "^__.*"      # 排除内部 topic
kafka_mm2_replication_factor: "3"
kafka_mm2_tasks_max: "3"

# 如果目标集群需要 SASL/SSL 认证
kafka_mm2_target_security_protocol: "SASL_SSL"
kafka_mm2_target_sasl_mechanism: "PLAIN"
kafka_mm2_target_sasl_jaas_config: 'org.apache.kafka.common.security.plain.PlainLoginModule required username="alikafka_xxx" password="xxx";'
kafka_mm2_target_ssl_truststore_location: "/etc/kafka/secrets/client.truststore.jks"

# 本地 truststore 文件路径（会自动复制到容器）
kafka_mm2_ssl_truststore_src: "/path/to/your/client.truststore.jks"
```

### 4. 部署 Kafka

#### 预检查
```bash
kolla-ansible -i ansible/inventory/multinode prechecks --tags kafka
```

#### 拉取镜像
```bash
kolla-ansible -i ansible/inventory/multinode pull --tags kafka
```

#### 部署
```bash
kolla-ansible -i ansible/inventory/multinode deploy --tags kafka
```

#### 重新配置
```bash
kolla-ansible -i ansible/inventory/multinode reconfigure --tags kafka
```

#### 停止服务
```bash
kolla-ansible -i ansible/inventory/multinode stop --tags kafka
```

### 5. 验证部署

#### 检查容器状态
```bash
docker ps | grep kafka
```

应该看到以下容器：
- `kafka_broker`
- `kafka_connect`
- `kafka_mm2`
- `kafka_ui`

#### 访问 Kafka UI
```
http://<your-host-ip>:8080
用户名: admin
密码: your-secure-password
```

#### 测试 Kafka Broker
```bash
# 进入 Kafka Broker 容器
docker exec -it kafka_broker bash

# 创建 topic
kafka-topics.sh --create --topic test --bootstrap-server localhost:9092 --partitions 3 --replication-factor 3

# 列出 topics
kafka-topics.sh --list --bootstrap-server localhost:9092

# 生产消息
echo "test message" | kafka-console-producer.sh --broker-list localhost:9092 --topic test

# 消费消息
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
```

#### 测试 Kafka Connect
```bash
# 查看 Connect 状态
curl http://<your-host-ip>:8083/

# 查看已安装的 connectors
curl http://<your-host-ip>:8083/connector-plugins
```

#### 测试 MirrorMaker 2
```bash
# 查看 MM2 运行状态
curl http://<your-host-ip>:8084/connectors

# 查看同步任务
curl http://<your-host-ip>:8084/connectors/MirrorSourceConnector/status
```

## 目录结构

```
ansible/
├── roles/
│   └── kafka/
│       ├── defaults/
│       │   └── main.yml              # 默认变量
│       ├── handlers/
│       │   └── main.yml              # 处理器（重启容器）
│       ├── tasks/
│       │   ├── main.yml              # 主任务入口
│       │   ├── config.yml            # 配置任务
│       │   ├── deploy.yml            # 部署任务
│       │   ├── check-containers.yml  # 容器检查
│       │   ├── deploy-containers.yml # 容器部署
│       │   ├── precheck.yml          # 预检查
│       │   ├── pull.yml              # 拉取镜像
│       │   ├── reconfigure.yml       # 重新配置
│       │   ├── stop.yml              # 停止服务
│       │   └── upgrade.yml           # 升级
│       ├── templates/
│       │   ├── kafka-broker.json.j2           # Broker 容器配置
│       │   ├── kafka-connect.json.j2          # Connect 容器配置
│       │   ├── kafka-mm2.json.j2              # MM2 容器配置
│       │   ├── kafka-ui.json.j2               # UI 容器配置
│       │   ├── server.properties.j2           # Kafka Broker 配置
│       │   ├── connect-distributed.properties.j2  # Connect 配置
│       │   ├── mm2.properties.j2              # MirrorMaker 2 配置
│       │   ├── log4j.properties.j2            # 日志配置
│       │   └── connect-log4j.properties.j2    # Connect 日志配置
│       └── vars/
│           └── main.yml              # 变量
├── kafka.yml                         # Kafka playbook
├── site.yml                          # 主 playbook（已集成 Kafka）
├── inventory/
│   ├── all-in-one                    # 单节点配置（已更新）
│   └── multinode                     # 多节点配置（已更新）
└── group_vars/
    └── kafka.yml                     # Kafka 组变量示例
```

## 配置文件说明

### Kafka Broker (server.properties)
- 支持 KRaft 模式（无需 Zookeeper）
- 自动生成 broker.id
- 配置复制因子和最小同步副本
- 日志保留策略
- 性能优化参数

### Kafka Connect (connect-distributed.properties)
- 分布式模式
- JSON 转换器
- 内部 topic 配置
- REST API 配置
- 插件路径

### MirrorMaker 2 (mm2.properties)
- 基于您提供的配置模板
- 支持 SASL/SSL 认证
- 单向数据同步（source -> destination）
- 性能优化参数
- IdentityReplicationPolicy（保持原 topic 名称）

### Kafka UI
- Web 管理界面
- 支持表单登录
- 可配置用户名密码

## 性能调优建议

### Broker
```yaml
# JVM 设置（根据服务器内存调整）
kafka_broker_heap_opts: "-Xms8G -Xmx8G -XX:+UseG1GC"

# 网络线程
num.network.threads: 8
num.io.threads: 16

# 日志配置
log.segment.bytes: 1073741824  # 1GB
```

### MirrorMaker 2
```yaml
# 任务数
kafka_mm2_tasks_max: 6

# 消费者配置
kafka_mm2_consumer_fetch_max_bytes: 67108864
kafka_mm2_consumer_max_poll_records: 5000

# 生产者配置
kafka_mm2_producer_batch_size: 32768
kafka_mm2_producer_linger_ms: 10
```

## 监控

Kafka 日志位置：
- Broker: `/var/log/kolla/kafka/`
- Connect: `/var/log/kolla/kafka-connect/`
- MM2: `/var/log/kolla/kafka-mm2/`
- UI: `/var/log/kolla/kafka-ui/`

JMX 端口：
- Broker: 9999

## 故障排查

### Broker 无法启动
```bash
# 查看日志
docker logs kafka_broker

# 检查数据目录权限
docker exec kafka_broker ls -la /var/lib/kafka/data/
```

### MirrorMaker 2 无法连接目标集群
```bash
# 检查 SSL 证书
docker exec kafka_mm2 ls -la /etc/kafka/secrets/

# 测试网络连接
docker exec kafka_mm2 telnet <target-broker> 9093
```

### Kafka UI 无法访问
```bash
# 检查端口
netstat -tlnp | grep 8080

# 查看环境变量
docker exec kafka_ui env | grep KAFKA
```

## 升级

```bash
# 更新镜像标签
# 在 globals.yml 中：
openstack_tag: "2024.1"

# 执行升级
kolla-ansible -i ansible/inventory/multinode upgrade --tags kafka
```

## 备份与恢复

### 备份配置
```bash
tar -czf kafka-config-backup.tar.gz /etc/kolla/kafka-*
```

### 备份数据
```bash
# 使用 Docker volume 备份
docker run --rm -v kafka_data:/data -v /backup:/backup alpine tar -czf /backup/kafka-data.tar.gz /data
```

## 安全建议

1. **修改默认密码**：更改 Kafka UI 管理员密码
2. **启用 TLS**：在生产环境中启用 SSL/TLS
3. **SASL 认证**：配置 SASL 认证机制
4. **网络隔离**：使用防火墙限制访问
5. **定期更新**：及时更新 Kafka 版本

## 支持的版本

- Kolla-Ansible: 2023.2 (Zed) 或 2024.1 (Caracal)
- Kafka: 3.x+
- Docker: 20.10+

## 相关链接

- [Apache Kafka 官方文档](https://kafka.apache.org/documentation/)
- [Kafka Connect 文档](https://kafka.apache.org/documentation/#connect)
- [MirrorMaker 2 文档](https://kafka.apache.org/documentation/#georeplication)
- [Kolla-Ansible 官方文档](https://docs.openstack.org/kolla-ansible/)

## 贡献

如有问题或建议，请提交 Issue 或 Pull Request。
