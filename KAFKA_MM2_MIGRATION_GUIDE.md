# Kafka MirrorMaker 2 迁移指南

## 将现有 MM2 配置迁移到 Kolla-Ansible

本指南帮助您将现有的 MirrorMaker 2 配置迁移到 Kolla-Ansible 部署方案。

## 原始配置对比

### 您的原始配置
```properties
# 原始 mm2.properties
clusters = source,destination

source.bootstrap.servers = 10.220.244.176:9092,10.220.244.177:9092,10.220.244.178:9092
destination.bootstrap.servers = 10.220.57.24:9092,10.220.57.25:9092,10.220.57.26:9092

source->destination.enabled = true
source->destination.topics = .*
destination->source.enabled = true
```

### 迁移后的 Kolla-Ansible 配置

在 `/etc/kolla/globals.yml` 或 `ansible/group_vars/kafka.yml` 中：

```yaml
# 启用 MirrorMaker 2
enable_kafka_mm2: "yes"

# 集群名称
kafka_mm2_source_cluster: "source"
kafka_mm2_target_cluster: "destination"

# 源集群配置
kafka_mm2_source_bootstrap_servers: "10.220.244.176:9092,10.220.244.177:9092,10.220.244.178:9092"
kafka_mm2_source_security_protocol: "PLAINTEXT"

# 目标集群配置
kafka_mm2_target_bootstrap_servers: "10.220.57.24:9092,10.220.57.25:9092,10.220.57.26:9092"
kafka_mm2_target_security_protocol: "PLAINTEXT"

# 同步配置
kafka_mm2_topics_pattern: ".*"
kafka_mm2_topics_exclude: "^__.*"
kafka_mm2_replication_enabled: "true"
kafka_mm2_replication_factor: "3"

# 性能配置
kafka_mm2_tasks_max: "8"
kafka_mm2_consumer_fetch_max_bytes: "67108864"
kafka_mm2_consumer_max_poll_records: "5000"
kafka_mm2_producer_batch_size: "33554432"
kafka_mm2_producer_linger_ms: "100"
kafka_mm2_producer_compression_type: "snappy"

# 禁用反向同步（如果需要单向同步）
# 注意：在 Kolla-Ansible 配置中，我们只配置一个方向的同步
# 如果需要双向同步，需要部署两个 MM2 实例
```

## 参考配置示例（来自您的需求）

### 示例 1：阿里云 Kafka 同步

```yaml
# 源集群（k8s 内部）
kafka_mm2_source_cluster: "kafka-unite-aidc-2"
kafka_mm2_source_bootstrap_servers: "kafka-b-controller-0.kafka-b-controller-headless.kafka.svc.cluster.local:9094,kafka-b-controller-1.kafka-b-controller-headless.kafka.svc.cluster.local:9094,kafka-b-controller-2.kafka-b-controller-headless.kafka.svc.cluster.local:9094"
kafka_mm2_source_security_protocol: "PLAINTEXT"

# 目标集群（阿里云公网）
kafka_mm2_target_cluster: "kafka-unite-cloud"
kafka_mm2_target_bootstrap_servers: "alikafka-post-cn-sco3upg0z006-1.alikafka.aliyuncs.com:9093,alikafka-post-cn-sco3upg0z006-2.alikafka.aliyuncs.com:9093,alikafka-post-cn-sco3upg0z006-3.alikafka.aliyuncs.com:9093"
kafka_mm2_target_security_protocol: "SASL_SSL"
kafka_mm2_target_sasl_mechanism: "PLAIN"
kafka_mm2_target_sasl_jaas_config: 'org.apache.kafka.common.security.plain.PlainLoginModule required username="alikafka_post-cn-sco3upg0z006" password="sYlfaYM8X6GSRkESlURP8oim6eKg7eos";'
kafka_mm2_target_ssl_truststore_location: "/etc/kafka/secrets/only.4096.client.truststore.jks"
kafka_mm2_target_ssl_endpoint_algorithm: ""

# SSL 证书文件（本地路径）
kafka_mm2_ssl_truststore_src: "/path/to/local/only.4096.client.truststore.jks"

# 同步特定 topic
kafka_mm2_topics_pattern: "model-token-usage"
kafka_mm2_topics_exclude: "^$"  # 不排除任何 topic

# 复制因子
kafka_mm2_replication_factor: "3"

# 性能优化
kafka_mm2_tasks_max: "3"
kafka_mm2_producer_batch_size: "16384"
kafka_mm2_producer_linger_ms: "1"
kafka_mm2_group_id: "unite-2-mirror-maker-2"
```

## 配置文件映射

### 原始配置项 → Kolla-Ansible 变量

| 原始配置 | Kolla-Ansible 变量 |
|---------|-------------------|
| `clusters` | `kafka_mm2_source_cluster`, `kafka_mm2_target_cluster` |
| `source.bootstrap.servers` | `kafka_mm2_source_bootstrap_servers` |
| `destination.bootstrap.servers` | `kafka_mm2_target_bootstrap_servers` |
| `source->destination.enabled` | `kafka_mm2_replication_enabled` |
| `source->destination.topics` | `kafka_mm2_topics_pattern` |
| `source->destination.topics.exclude` | `kafka_mm2_topics_exclude` |
| `replication.factor` | `kafka_mm2_replication_factor` |
| `tasks.max` | `kafka_mm2_tasks_max` |
| `group.id` | `kafka_mm2_group_id` |
| `producer.batch.size` | `kafka_mm2_producer_batch_size` |
| `producer.linger.ms` | `kafka_mm2_producer_linger_ms` |
| `consumer.fetch.max.bytes` | `kafka_mm2_consumer_fetch_max_bytes` |

## 完整的配置示例

创建 `/etc/kolla/config/kafka-mm2/mm2-custom.yml`（可选的自定义配置）：

```yaml
# 如果需要更复杂的配置，可以直接覆盖 mm2.properties
# 这个文件会在部署时被复制到容器中

# 例如：添加自定义配置
kafka_mm2_custom_config: |
  # 自定义的 MirrorMaker 2 配置
  refresh.topics.enabled=true
  refresh.topics.interval.seconds=600
  sync.topic.configs.interval.seconds=600
```

## 部署步骤

### 1. 准备 SSL 证书（如果需要）

```bash
# 将 truststore 文件放在可访问的位置
mkdir -p /opt/kafka-certs
cp only.4096.client.truststore.jks /opt/kafka-certs/
chmod 644 /opt/kafka-certs/only.4096.client.truststore.jks
```

### 2. 配置 globals.yml

```yaml
# /etc/kolla/globals.yml

# 启用 MM2
enable_kafka_mm2: "yes"

# 基础配置（如上所述）
kafka_mm2_source_cluster: "source"
kafka_mm2_target_cluster: "destination"
# ... 其他配置
```

### 3. 配置 inventory

```ini
# 指定 MM2 运行的节点
[kafka-mm2]
kafka01
```

### 4. 部署

```bash
# 仅部署 MM2
kolla-ansible -i /path/to/inventory deploy --tags kafka-mm2

# 或者部署整个 Kafka 生态系统
kolla-ansible -i /path/to/inventory deploy --tags kafka
```

### 5. 验证

```bash
# 检查 MM2 容器
docker ps | grep kafka_mm2

# 查看 MM2 日志
docker logs kafka_mm2

# 检查连接器状态
curl http://localhost:8084/connectors
curl http://localhost:8084/connectors/MirrorSourceConnector/status

# 验证 topic 同步
# 在目标集群上检查 topic
kafka-topics.sh --bootstrap-server <destination-broker>:9092 --list
```

## 监控和调试

### 查看同步进度

```bash
# 进入 MM2 容器
docker exec -it kafka_mm2 bash

# 检查 connector 配置
curl -s http://localhost:8084/connectors/MirrorSourceConnector | jq

# 查看任务状态
curl -s http://localhost:8084/connectors/MirrorSourceConnector/tasks | jq

# 查看 metrics
curl -s http://localhost:8084/connectors/MirrorSourceConnector/metrics | jq
```

### 常见问题排查

#### 1. 无法连接到源或目标集群

```bash
# 测试网络连通性
docker exec kafka_mm2 telnet source-broker 9092
docker exec kafka_mm2 telnet destination-broker 9092

# 检查 DNS 解析
docker exec kafka_mm2 nslookup source-broker
```

#### 2. SSL 认证失败

```bash
# 检查 truststore 文件
docker exec kafka_mm2 ls -la /etc/kafka/secrets/
docker exec kafka_mm2 keytool -list -keystore /etc/kafka/secrets/client.truststore.jks

# 查看详细错误日志
docker exec kafka_mm2 tail -f /var/log/kolla/kafka-mm2/connect.log
```

#### 3. Topic 没有同步

```bash
# 检查 topic 正则表达式
# 确保 kafka_mm2_topics_pattern 匹配您的 topic

# 查看 MirrorSourceConnector 配置
curl http://localhost:8084/connectors/MirrorSourceConnector/config | jq
```

## 高级配置

### 双向同步

如果需要双向同步，您需要：

1. 部署两个 MM2 实例（两个不同的容器或节点）
2. 第一个实例：source -> destination
3. 第二个实例：destination -> source

或者修改配置启用双向：

```yaml
# 在 kafka_mm2_custom_config 中添加
kafka_mm2_custom_config: |
  {{ kafka_mm2_target_cluster }}->{{ kafka_mm2_source_cluster }}.enabled=true
  {{ kafka_mm2_target_cluster }}->{{ kafka_mm2_source_cluster }}.topics=.*
```

### 过滤特定 Topic

```yaml
# 只同步特定的 topics
kafka_mm2_topics_pattern: "important-topic-.*|critical-data"

# 排除敏感 topics
kafka_mm2_topics_exclude: "^__.*|sensitive-.*|private-.*"
```

### 性能优化

```yaml
# 增加并发任务
kafka_mm2_tasks_max: "12"

# 优化消费者
kafka_mm2_consumer_fetch_max_bytes: "104857600"  # 100MB
kafka_mm2_consumer_max_poll_records: "10000"

# 优化生产者
kafka_mm2_producer_batch_size: "65536"  # 64KB
kafka_mm2_producer_linger_ms: "50"
kafka_mm2_producer_max_request_size: "104857600"  # 100MB
```

## 迁移检查清单

- [ ] SSL 证书已准备并放在正确位置
- [ ] 网络连通性已验证（源集群和目标集群）
- [ ] Topic 名称和正则表达式已确认
- [ ] 认证信息（用户名/密码）已正确配置
- [ ] 复制因子符合目标集群配置
- [ ] 资源配额（内存、CPU）已规划
- [ ] 监控和告警已配置
- [ ] 回滚方案已准备

## 回滚方案

如果需要回滚到原始部署：

```bash
# 停止 Kolla-Ansible 部署的 MM2
kolla-ansible -i /path/to/inventory stop --tags kafka-mm2

# 恢复原始配置和启动方式
# 使用原始的 connect-mirror-maker.sh 脚本
```

## 参考资源

- [MirrorMaker 2 官方文档](https://cwiki.apache.org/confluence/display/KAFKA/KIP-382%3A+MirrorMaker+2.0)
- [Kafka Connect 配置](https://kafka.apache.org/documentation/#connectconfigs)
- [阿里云 Kafka 文档](https://help.aliyun.com/product/68151.html)
