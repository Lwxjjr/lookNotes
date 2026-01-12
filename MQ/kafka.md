docker-compose.yml
```yml
version: '3.8'

services:
  kafka:
    image: bitnami/kafka:3.7
    container_name: kafka
    hostname: kafka
    ports:
      - "9092:9092"
    environment:
      # Bitnami 特定的 KRaft 配置
      KAFKA_CFG_NODE_ID: 1
      KAFKA_CFG_PROCESS_ROLES: "broker,controller"
      KAFKA_CFG_CONTROLLER_LISTENER_NAMES: "CONTROLLER"
      
      # 监听器配置
      KAFKA_CFG_LISTENERS: "PLAINTEXT://:9092,CONTROLLER://:9093"
      KAFKA_CFG_ADVERTISED_LISTENERS: "PLAINTEXT://kafka:9092"
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: "CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT"
      
      # 仲裁配置
      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: "1@kafka:9093"
      
      # 集群配置
      KAFKA_CFG_NUM_PARTITIONS: 1
      KAFKA_CFG_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_CFG_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_CFG_TRANSACTION_STATE_LOG_MIN_ISR: 1
      
      # 禁用 ZooKeeper
      KAFKA_CFG_ZOOKEEPER_CONNECT: ""
      ALLOW_PLAINTEXT_LISTENER: "yes"

    volumes:
      - kafka_data:/bitnami/kafka

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: "kraft-kafka"
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: "kafka:9092"
    depends_on:
      - kafka

volumes:
  kafka_data:
```

