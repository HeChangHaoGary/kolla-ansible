# Kafka Docker 镜像构建指南

## 概述

为了在 Kolla-Ansible 中使用 Kafka，您需要构建相应的 Docker 镜像。本文档提供镜像构建的指南。

## 方案一：使用现有的 Kafka 镜像（推荐）

### 1. 使用官方 Confluent 镜像

修改 `ansible/group_vars/kafka.yml`：

```yaml
# 使用 Confluent Platform 镜像
kafka_broker_image: "confluentinc/cp-kafka"
kafka_broker_tag: "7.5.0"

kafka_connect_image: "confluentinc/cp-kafka-connect"
kafka_connect_tag: "7.5.0"

kafka_mm2_image: "confluentinc/cp-kafka"
kafka_mm2_tag: "7.5.0"
```

### 2. 使用 Bitnami 镜像

```yaml
kafka_broker_image: "bitnami/kafka"
kafka_broker_tag: "3.6.0"

kafka_connect_image: "bitnami/kafka"
kafka_connect_tag: "3.6.0"

kafka_mm2_image: "bitnami/kafka"
kafka_mm2_tag: "3.6.0"
```

## 方案二：构建 Kolla 风格的 Kafka 镜像

### 1. 创建 Dockerfile

创建 `docker/kafka/Dockerfile.j2`：

```dockerfile
FROM {{ base_distro }}:{{ base_distro_tag }}

LABEL maintainer="Kolla Project (https://kolla.io)"

{% block kafka_header %}{% endblock %}

{% import "macros.j2" as macros with context %}

{{ macros.configure_user(name='kafka', groups='kolla') }}

{% if install_type == 'binary' %}
    {% if base_distro in ['ubuntu', 'debian'] %}

RUN apt-get update \
    && apt-get -y install --no-install-recommends \
        openjdk-11-jre-headless \
        wget \
        ca-certificates \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

    {% elif base_distro in ['rocky', 'centos'] %}

RUN dnf -y install \
        java-11-openjdk-headless \
        wget \
        ca-certificates \
    && dnf clean all \
    && rm -rf /var/cache/dnf

    {% endif %}

# Download and install Kafka
ARG KAFKA_VERSION=3.6.0
ARG SCALA_VERSION=2.13

RUN wget -q https://archive.apache.org/dist/kafka/${KAFKA_VERSION}/kafka_${SCALA_VERSION}-${KAFKA_VERSION}.tgz \
    && tar -xzf kafka_${SCALA_VERSION}-${KAFKA_VERSION}.tgz \
    && mv kafka_${SCALA_VERSION}-${KAFKA_VERSION} /opt/kafka \
    && rm kafka_${SCALA_VERSION}-${KAFKA_VERSION}.tgz \
    && mkdir -p /opt/kafka/plugins \
    && chown -R kafka:kafka /opt/kafka

{% elif install_type == 'source' %}
# TODO: Add source installation method
{% endif %}

# Create necessary directories
RUN mkdir -p /var/lib/kafka/data \
    && mkdir -p /var/lib/kafka/connect \
    && mkdir -p /var/lib/kafka/mm2 \
    && mkdir -p /var/log/kolla/kafka \
    && mkdir -p /etc/kafka/secrets \
    && chown -R kafka:kafka /var/lib/kafka \
    && chown -R kafka:kafka /var/log/kolla/kafka \
    && chown -R kafka:kafka /etc/kafka

# Add healthcheck scripts
COPY kafka_extend_start.sh /usr/local/bin/kolla_kafka_extend_start
COPY healthcheck_* /usr/local/bin/

RUN chmod 755 /usr/local/bin/kolla_kafka_extend_start \
    && chmod 755 /usr/local/bin/healthcheck_*

ENV PATH="/opt/kafka/bin:${PATH}"

USER kafka

{% block kafka_footer %}{% endblock %}
{% block footer %}{% endblock %}
```

### 2. 创建启动脚本

创建 `docker/kafka/kafka_extend_start.sh`：

```bash
#!/bin/bash

# This file is used by Kolla to do configuration

set -o errexit

# Loading common functions.
source /usr/local/bin/kolla_httpd_setup

# Execute config strategy
set_configs

# Bootstrap Kolla
bootstrap_kolla

# Run command
exec "$@"
```

### 3. 使用 Kolla 构建工具构建

```bash
# 安装 kolla
pip install kolla

# 构建 Kafka 镜像
kolla-build kafka \
    --base rocky \
    --base-tag 9 \
    --namespace your-registry/kolla \
    --tag 2024.1 \
    --registry your-registry.com
```

## 方案三：手动构建简化版镜像

### Kafka Broker Dockerfile

```dockerfile
FROM eclipse-temurin:17-jre

# 创建 kafka 用户
RUN groupadd -r kafka && useradd -r -g kafka kafka

# 下载并安装 Kafka
ARG KAFKA_VERSION=3.6.0
ARG SCALA_VERSION=2.13

RUN apt-get update && \
    apt-get install -y wget && \
    wget -q https://archive.apache.org/dist/kafka/${KAFKA_VERSION}/kafka_${SCALA_VERSION}-${KAFKA_VERSION}.tgz && \
    tar -xzf kafka_${SCALA_VERSION}-${KAFKA_VERSION}.tgz && \
    mv kafka_${SCALA_VERSION}-${KAFKA_VERSION} /opt/kafka && \
    rm kafka_${SCALA_VERSION}-${KAFKA_VERSION}.tgz && \
    apt-get remove -y wget && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 创建目录
RUN mkdir -p /var/lib/kafka/data \
    && mkdir -p /var/log/kolla/kafka \
    && chown -R kafka:kafka /opt/kafka \
    && chown -R kafka:kafka /var/lib/kafka \
    && chown -R kafka:kafka /var/log/kolla/kafka

WORKDIR /opt/kafka

USER kafka

ENV PATH="/opt/kafka/bin:${PATH}"

EXPOSE 9092 9093

CMD ["kafka-server-start.sh", "/opt/kafka/config/server.properties"]
```

### 构建命令

```bash
# 构建 Kafka 镜像
docker build -t your-registry/kolla/kafka:2024.1 -f Dockerfile.kafka .

# 推送到镜像仓库
docker push your-registry/kolla/kafka:2024.1

# 在 globals.yml 中配置
docker_registry: "your-registry"
docker_namespace: "kolla"
openstack_tag: "2024.1"
```

## 镜像配置要求

无论使用哪种方案，镜像都需要满足以下要求：

### 必需的二进制文件
- `/opt/kafka/bin/kafka-server-start.sh` - Kafka Broker 启动脚本
- `/opt/kafka/bin/connect-distributed.sh` - Connect 启动脚本
- `/opt/kafka/bin/connect-mirror-maker.sh` - MirrorMaker 2 启动脚本
- `/opt/kafka/bin/kafka-topics.sh` - Topic 管理工具
- `/opt/kafka/bin/kafka-console-producer.sh` - 生产者工具
- `/opt/kafka/bin/kafka-console-consumer.sh` - 消费者工具

### 必需的目录
- `/opt/kafka/config/` - 配置文件目录
- `/var/lib/kafka/data/` - 数据目录
- `/var/log/kolla/kafka/` - 日志目录
- `/opt/kafka/plugins/` - Connect 插件目录

### 必需的用户
- `kafka` 用户和组（UID/GID 建议：42465:42465）

## Kafka Connect 插件安装

如果需要使用 Kafka Connect 的额外插件：

```bash
# 下载插件到本地
mkdir -p /opt/kafka-plugins
cd /opt/kafka-plugins
wget https://example.com/kafka-connect-jdbc.zip
unzip kafka-connect-jdbc.zip

# 在部署前将插件复制到目标主机
ansible kafka-connect -m copy -a "src=/opt/kafka-plugins/ dest=/var/lib/docker/volumes/kafka_connect_plugins/_data/"
```

或者在 `ansible/group_vars/kafka.yml` 中配置：

```yaml
kafka_connect_extra_volumes:
  - "/path/to/local/plugins:/opt/kafka/plugins:ro"
```

## MirrorMaker 2 SSL 证书

如果目标集群需要 SSL 认证：

```bash
# 准备 truststore 文件
# 假设您已经有了 client.truststore.jks

# 在 globals.yml 中配置
kafka_mm2_ssl_truststore_src: "/path/to/local/client.truststore.jks"
kafka_mm2_target_ssl_truststore_location: "/etc/kafka/secrets/client.truststore.jks"

# Kolla 会自动将文件复制到容器中
```

## 验证镜像

```bash
# 测试 Kafka 镜像
docker run --rm your-registry/kolla/kafka:2024.1 kafka-broker-api-versions.sh --version

# 测试 Connect 镜像
docker run --rm your-registry/kolla/kafka:2024.1 connect-distributed.sh --version

# 检查目录结构
docker run --rm your-registry/kolla/kafka:2024.1 ls -la /opt/kafka/
```

## 推荐的镜像仓库配置

### 使用阿里云容器镜像服务

```yaml
# globals.yml
docker_registry: "registry.cn-hangzhou.aliyuncs.com"
docker_namespace: "your-namespace"
docker_registry_username: "your-username"
docker_registry_password: "your-password"
```

### 使用私有 Harbor 仓库

```yaml
# globals.yml
docker_registry: "harbor.example.com"
docker_namespace: "kolla"
docker_registry_username: "admin"
docker_registry_password: "Harbor12345"
docker_registry_insecure: "no"  # 如果使用 HTTPS
```

## 常见问题

### Q: 镜像太大怎么办？
A: 使用多阶段构建或精简基础镜像：
```dockerfile
# 使用 alpine 作为基础镜像
FROM alpine:3.18
RUN apk add --no-cache openjdk17-jre
```

### Q: 如何更新 Kafka 版本？
A: 修改构建参数：
```bash
kolla-build kafka --build-args KAFKA_VERSION=3.7.0
```

### Q: Connect 插件在哪里？
A: 插件应该放在 `/opt/kafka/plugins/` 目录，该目录挂载为 Docker volume。

## 后续步骤

镜像构建完成后，请参考 `KAFKA_DEPLOYMENT_GUIDE.md` 进行部署。
