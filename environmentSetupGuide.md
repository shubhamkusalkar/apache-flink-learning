# Local Flink + Kafka Development Environment Setup Guide

## Overview
This guide provides multiple ways to set up Flink and Kafka locally for practice. Choose the method that best fits your needs.

**Setup Options**:
1. **Docker Compose** (Recommended - Easiest, most flexible)
2. **Manual Installation** (Best for understanding internals)
3. **Kubernetes (Minikube)** (For production-like practice)

---

## Option 1: Docker Compose Setup (RECOMMENDED)

### Why Docker Compose?
✅ Quick setup (5 minutes)\
✅ Easy to start/stop/reset\
✅ Matches production configuration\
✅ Includes monitoring tools\
✅ No system pollution

### Prerequisites
```bash
# Install Docker Desktop (includes Docker Compose)
# Windows/Mac: Download from https://www.docker.com/products/docker-desktop
# Linux:
sudo apt-get update
sudo apt-get install docker.io docker-compose
sudo usermod -aG docker $USER  # Add yourself to docker group
# Log out and back in
```

### Step 1: Create Project Structure
```bash
mkdir flink-kafka-practice
cd flink-kafka-practice
mkdir -p {flink-jobs,kafka-data,flink-checkpoints,flink-savepoints}
```

### Step 2: Create docker-compose.yml

Create a file named `docker-compose.yml`:

```yaml
version: '3.8'

services:
  #######################################
  # Zookeeper (required for Kafka)
  #######################################
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
      - zookeeper-logs:/var/lib/zookeeper/log

  #######################################
  # Kafka Broker
  #######################################
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    hostname: kafka
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"      # External access
      - "19092:19092"    # Internal Docker network
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:19092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
      KAFKA_LOG4J_LOGGERS: "kafka.controller=INFO,kafka.producer.async.DefaultEventHandler=INFO,state.change.logger=INFO"
    volumes:
      - kafka-data:/var/lib/kafka/data

  #######################################
  # Schema Registry (for Avro support)
  #######################################
  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    hostname: schema-registry
    container_name: schema-registry
    depends_on:
      - kafka
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: 'kafka:19092'
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081

  #######################################
  # Kafka UI (Web interface)
  #######################################
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    depends_on:
      - kafka
      - schema-registry
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:19092
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
      DYNAMIC_CONFIG_ENABLED: 'true'

  #######################################
  # Flink JobManager
  #######################################
  flink-jobmanager:
    image: flink:1.18.0-scala_2.12-java11
    hostname: flink-jobmanager
    container_name: flink-jobmanager
    ports:
      - "8081:8081"  # Web UI
    command: jobmanager
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: flink-jobmanager
        state.backend: rocksdb
        state.backend.incremental: true
        state.checkpoints.dir: file:///tmp/flink-checkpoints
        state.savepoints.dir: file:///tmp/flink-savepoints
        execution.checkpointing.interval: 60s
        execution.checkpointing.mode: EXACTLY_ONCE
        rest.flamegraph.enabled: true
    volumes:
      - ./flink-checkpoints:/tmp/flink-checkpoints
      - ./flink-savepoints:/tmp/flink-savepoints
      - ./flink-jobs:/opt/flink/usrlib

  #######################################
  # Flink TaskManager
  #######################################
  flink-taskmanager:
    image: flink:1.18.0-scala_2.12-java11
    hostname: flink-taskmanager
    container_name: flink-taskmanager
    depends_on:
      - flink-jobmanager
    command: taskmanager
    scale: 2  # Run 2 TaskManagers
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: flink-jobmanager
        taskmanager.numberOfTaskSlots: 4
        taskmanager.memory.process.size: 2048m
        state.backend: rocksdb
        state.backend.incremental: true
    volumes:
      - ./flink-checkpoints:/tmp/flink-checkpoints
      - ./flink-savepoints:/tmp/flink-savepoints

  #######################################
  # PostgreSQL (for practice with JDBC sink)
  #######################################
  postgres:
    image: postgres:15
    container_name: postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: flink_practice
      POSTGRES_USER: flink
      POSTGRES_PASSWORD: flink123
    volumes:
      - postgres-data:/var/lib/postgresql/data

  #######################################
  # Elasticsearch (for practice with ES sink)
  #######################################
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data

  #######################################
  # Kibana (for visualizing ES data)
  #######################################
  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    depends_on:
      - elasticsearch
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200

volumes:
  zookeeper-data:
  zookeeper-logs:
  kafka-data:
  postgres-data:
  elasticsearch-data:

networks:
  default:
    name: flink-kafka-network
```

### Step 3: Start the Environment
```bash
# Start all services
docker-compose up -d

# Check if all services are running
docker-compose ps

# View logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f kafka
docker-compose logs -f flink-jobmanager
```

### Step 4: Verify Installation
```bash
# Test Kafka
docker exec -it kafka kafka-topics --list --bootstrap-server localhost:9092

# Create a test topic
docker exec -it kafka kafka-topics --create \
  --topic test-topic \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1

# Produce test messages
docker exec -it kafka kafka-console-producer \
  --topic test-topic \
  --bootstrap-server localhost:9092
# Type some messages and press Ctrl+C to exit

# Consume messages
docker exec -it kafka kafka-console-consumer \
  --topic test-topic \
  --from-beginning \
  --bootstrap-server localhost:9092
```

### Step 5: Access Web UIs
- **Flink Dashboard**: http://localhost:8081
- **Kafka UI**: http://localhost:8080
- **Kibana**: http://localhost:5601
- **Schema Registry**: http://localhost:8081 (schema-registry)

### Step 6: Create Your First Flink Job

Create `flink-jobs/KafkaFlinkExample.java`:

```java
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.connector.kafka.source.KafkaSource;
import org.apache.flink.connector.kafka.source.enumerator.initializer.OffsetsInitializer;
import org.apache.flink.connector.kafka.sink.KafkaRecordSerializationSchema;
import org.apache.flink.connector.kafka.sink.KafkaSink;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class KafkaFlinkExample {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        
        KafkaSource<String> source = KafkaSource.<String>builder()
            .setBootstrapServers("kafka:19092")
            .setTopics("input-topic")
            .setGroupId("flink-group")
            .setStartingOffsets(OffsetsInitializer.earliest())
            .setValueOnlyDeserializer(new SimpleStringSchema())
            .build();
        
        DataStream<String> stream = env.fromSource(
            source, 
            WatermarkStrategy.noWatermarks(), 
            "Kafka Source"
        );
        
        DataStream<String> processed = stream.map(String::toUpperCase);
        
        KafkaSink<String> sink = KafkaSink.<String>builder()
            .setBootstrapServers("kafka:19092")
            .setRecordSerializer(
                KafkaRecordSerializationSchema.builder()
                    .setTopic("output-topic")
                    .setValueSerializationSchema(new SimpleStringSchema())
                    .build()
            )
            .build();
        
        processed.sinkTo(sink);
        
        env.execute("Kafka Flink Example");
    }
}
```

### Step 7: Useful Commands

```bash
# Stop all services
docker-compose down

# Stop and remove all data (fresh start)
docker-compose down -v

# Restart a specific service
docker-compose restart kafka

# Scale TaskManagers
docker-compose up -d --scale flink-taskmanager=3

# Execute commands inside containers
docker exec -it flink-jobmanager bash
docker exec -it kafka bash

# Copy JAR to Flink
docker cp my-flink-job.jar flink-jobmanager:/opt/flink/usrlib/

# Submit Flink job
docker exec -it flink-jobmanager flink run /opt/flink/usrlib/my-flink-job.jar
```

---

## Option 2: Manual Installation (Understanding Internals)

### For Learning How Things Work Under the Hood

### Step 1: Install Java
```bash
# Check if Java 11 is installed
java -version

# If not, install (Ubuntu/Debian)
sudo apt update
sudo apt install openjdk-11-jdk

# macOS (using Homebrew)
brew install openjdk@11

# Windows: Download from https://adoptium.net/
```

### Step 2: Install Kafka

```bash
# Download Kafka
cd ~/Downloads
wget https://archive.apache.org/dist/kafka/3.5.1/kafka_2.13-3.5.1.tgz

# Extract
tar -xzf kafka_2.13-3.5.1.tgz
sudo mv kafka_2.13-3.5.1 /opt/kafka

# Add to PATH (add to ~/.bashrc or ~/.zshrc)
export KAFKA_HOME=/opt/kafka
export PATH=$PATH:$KAFKA_HOME/bin

# Create data directories
mkdir -p ~/kafka-data/zookeeper
mkdir -p ~/kafka-data/kafka-logs
```

### Step 3: Configure Kafka

Edit `/opt/kafka/config/zookeeper.properties`:
```properties
dataDir=/home/YOUR_USERNAME/kafka-data/zookeeper
clientPort=2181
maxClientCnxns=0
admin.enableServer=false
```

Edit `/opt/kafka/config/server.properties`:
```properties
broker.id=0
listeners=PLAINTEXT://localhost:9092
log.dirs=/home/YOUR_USERNAME/kafka-data/kafka-logs
num.partitions=3
log.retention.hours=168
zookeeper.connect=localhost:2181
```

### Step 4: Start Kafka

```bash
# Terminal 1: Start Zookeeper
zookeeper-server-start.sh $KAFKA_HOME/config/zookeeper.properties

# Terminal 2: Start Kafka Broker
kafka-server-start.sh $KAFKA_HOME/config/server.properties

# Test Kafka
kafka-topics.sh --create --topic test --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
kafka-topics.sh --list --bootstrap-server localhost:9092
```

### Step 5: Install Flink

```bash
# Download Flink
cd ~/Downloads
wget https://archive.apache.org/dist/flink/flink-1.18.0/flink-1.18.0-bin-scala_2.12.tgz

# Extract
tar -xzf flink-1.18.0-bin-scala_2.12.tgz
sudo mv flink-1.18.0 /opt/flink

# Add to PATH
export FLINK_HOME=/opt/flink
export PATH=$PATH:$FLINK_HOME/bin
```

### Step 6: Configure Flink

Edit `/opt/flink/conf/flink-conf.yaml`:
```yaml
jobmanager.rpc.address: localhost
jobmanager.memory.process.size: 1600m
taskmanager.memory.process.size: 2048m
taskmanager.numberOfTaskSlots: 4
parallelism.default: 2
state.backend: rocksdb
state.backend.incremental: true
state.checkpoints.dir: file:///tmp/flink-checkpoints
state.savepoints.dir: file:///tmp/flink-savepoints
```

### Step 7: Start Flink

```bash
# Start Flink cluster
start-cluster.sh

# Check status
jps  # Should see JobManager and TaskManager

# Access Web UI
# http://localhost:8081

# Stop Flink
stop-cluster.sh
```

### Step 8: Create Systemd Services (Linux - Optional)

Make Kafka and Flink start automatically:

`/etc/systemd/system/zookeeper.service`:
```ini
[Unit]
Description=Apache Zookeeper
After=network.target

[Service]
Type=simple
User=YOUR_USERNAME
ExecStart=/opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
ExecStop=/opt/kafka/bin/zookeeper-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

`/etc/systemd/system/kafka.service`:
```ini
[Unit]
Description=Apache Kafka
After=zookeeper.service
Requires=zookeeper.service

[Service]
Type=simple
User=YOUR_USERNAME
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

Enable services:
```bash
sudo systemctl daemon-reload
sudo systemctl enable zookeeper kafka
sudo systemctl start zookeeper kafka
sudo systemctl status zookeeper kafka
```

---

## Option 3: IntelliJ IDEA Setup (For Development)

### Step 1: Install IntelliJ IDEA
Download Community Edition from https://www.jetbrains.com/idea/download/

### Step 2: Create Maven Project

```bash
# Using Flink quickstart archetype
mvn archetype:generate \
  -DarchetypeGroupId=org.apache.flink \
  -DarchetypeArtifactId=flink-quickstart-java \
  -DarchetypeVersion=1.18.0 \
  -DgroupId=com.practice \
  -DartifactId=flink-kafka-practice \
  -Dversion=1.0-SNAPSHOT \
  -Dpackage=com.practice.flink \
  -DinteractiveMode=false

cd flink-kafka-practice
```

### Step 3: Update pom.xml

Add Kafka connector dependency:

```xml
<dependencies>
    <!-- Flink dependencies -->
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-streaming-java</artifactId>
        <version>1.18.0</version>
        <scope>provided</scope>
    </dependency>

    <!-- Kafka Connector -->
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-connector-kafka</artifactId>
        <version>3.0.1-1.18</version>
    </dependency>

    <!-- JSON support -->
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-json</artifactId>
        <version>1.18.0</version>
    </dependency>

    <!-- Avro support -->
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-avro</artifactId>
        <version>1.18.0</version>
    </dependency>

    <!-- Logging -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-simple</artifactId>
        <version>1.7.36</version>
    </dependency>

    <!-- Testing -->
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-test-utils</artifactId>
        <version>1.18.0</version>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>kafka</artifactId>
        <version>1.19.3</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>3.5.0</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### Step 4: Run from IntelliJ

1. Import the project into IntelliJ
2. Create a run configuration with VM options:
   ```
   -Dlog4j.configuration=file:src/main/resources/log4j.properties
   ```
3. Run your Flink jobs directly from IDE (uses embedded cluster)

---

## Quick Practice Scripts

### Script 1: Kafka Topic Manager
Create `kafka-manager.sh`:

```bash
#!/bin/bash

KAFKA_HOME=/opt/kafka
BOOTSTRAP_SERVER=localhost:9092

case "$1" in
  create)
    kafka-topics.sh --create \
      --topic $2 \
      --bootstrap-server $BOOTSTRAP_SERVER \
      --partitions ${3:-3} \
      --replication-factor ${4:-1}
    ;;
  list)
    kafka-topics.sh --list --bootstrap-server $BOOTSTRAP_SERVER
    ;;
  describe)
    kafka-topics.sh --describe --topic $2 --bootstrap-server $BOOTSTRAP_SERVER
    ;;
  delete)
    kafka-topics.sh --delete --topic $2 --bootstrap-server $BOOTSTRAP_SERVER
    ;;
  produce)
    kafka-console-producer.sh --topic $2 --bootstrap-server $BOOTSTRAP_SERVER
    ;;
  consume)
    kafka-console-consumer.sh --topic $2 --from-beginning --bootstrap-server $BOOTSTRAP_SERVER
    ;;
  *)
    echo "Usage: $0 {create|list|describe|delete|produce|consume} [topic-name] [partitions] [replication]"
    exit 1
    ;;
esac
```

Make it executable:
```bash
chmod +x kafka-manager.sh

# Usage examples
./kafka-manager.sh create my-topic 3 1
./kafka-manager.sh list
./kafka-manager.sh produce my-topic
./kafka-manager.sh consume my-topic
```

### Script 2: Flink Job Manager
Create `flink-manager.sh`:

```bash
#!/bin/bash

FLINK_HOME=/opt/flink

case "$1" in
  start)
    start-cluster.sh
    echo "Flink started. Access UI at http://localhost:8081"
    ;;
  stop)
    stop-cluster.sh
    ;;
  submit)
    flink run $2
    ;;
  list)
    flink list
    ;;
  cancel)
    flink cancel $2
    ;;
  savepoint)
    flink savepoint $2 $3
    ;;
  *)
    echo "Usage: $0 {start|stop|submit|list|cancel|savepoint} [args]"
    exit 1
    ;;
esac
```

---

## Practice Exercises

### Exercise 1: Hello World Pipeline
```bash
# 1. Create topics
docker exec -it kafka kafka-topics --create --topic input --bootstrap-server localhost:9092 --partitions 3
docker exec -it kafka kafka-topics --create --topic output --bootstrap-server localhost:9092 --partitions 3

# 2. Produce messages
docker exec -it kafka kafka-console-producer --topic input --bootstrap-server localhost:9092
# Type: hello, world, flink

# 3. Consume from output
docker exec -it kafka kafka-console-consumer --topic output --bootstrap-server localhost:9092 --from-beginning
```

### Exercise 2: Windowed Aggregation
- Read sensor data from Kafka
- Calculate average per 1-minute tumbling window
- Write results to output topic

### Exercise 3: Stream Joins
- Create two topics: orders and products
- Join streams in Flink
- Write enriched data to output

---

## Troubleshooting

### Problem: Kafka won't start
```bash
# Check if ports are in use
sudo lsof -i :2181  # Zookeeper
sudo lsof -i :9092  # Kafka

# Clear Kafka data and restart
rm -rf ~/kafka-data/*
# Restart Kafka and Zookeeper
```

### Problem: Flink job fails with connection error
```bash
# Check if Kafka is accessible
telnet localhost 9092

# Verify bootstrap server in your code matches:
# - Docker: kafka:19092 (internal) or localhost:9092 (external)
# - Manual: localhost:9092
```

### Problem: Out of memory
```bash
# Increase Docker memory (Docker Desktop -> Settings -> Resources)
# Or increase JVM heap in docker-compose.yml:
# taskmanager.memory.process.size: 4096m
```

### Problem: Consumer lag building up
```bash
# Check consumer group
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group your-group

# Increase Flink parallelism
# Increase Kafka partitions
```

---

## Daily Workflow

### Morning Setup (Docker Compose)
```bash
cd ~/flink-kafka-practice
docker-compose up -d
docker-compose ps  # Verify all services running
```

### During Practice
```bash
# Watch logs
docker-compose logs -f flink-jobmanager

# Access UIs
# Flink: http://localhost:8081
# Kafka: http://localhost:8080

# Create topics, produce/consume data
# Submit Flink jobs
# Monitor performance
```

### End of Day
```bash
# Option 1: Keep running (uses resources)
# No action needed

# Option 2: Stop but keep data
docker-compose stop

# Option 3: Clean everything
docker-compose down -v
```

---

## Recommended Setup

**For Beginners**: Docker Compose (easy, complete)
**For Learning Internals**: Manual installation
**For Development**: IntelliJ + Docker Compose
**For Production Practice**: Kubernetes (Minikube)

**Best Overall**: Docker Compose + IntelliJ IDEA

---

## Next Steps

1. **Today**: Set up Docker Compose environment
2. **Tomorrow**: Create your first topic and run WordCount
3. **This Week**: Build the 5 practice exercises
4. **Next Week**: Start working on real projects from the roadmap

---

## Quick Reference Card

### Kafka Commands (Docker)
```bash
# List topics
docker exec kafka kafka-topics --list --bootstrap-server localhost:9092

# Create topic
docker exec kafka kafka-topics --create --topic NAME --bootstrap-server localhost:9092 --partitions 3

# Produce
docker exec -it kafka kafka-console-producer --topic NAME --bootstrap-server localhost:9092

# Consume
docker exec -it kafka kafka-console-consumer --topic NAME --bootstrap-server localhost:9092 --from-beginning

# Consumer groups
docker exec kafka kafka-consumer-groups --bootstrap-server localhost:9092 --list
```

### Flink Commands (Docker)
```bash
# Submit job
docker cp myJob.jar flink-jobmanager:/tmp/
docker exec flink-jobmanager flink run /tmp/myJob.jar

# List jobs
docker exec flink-jobmanager flink list

# Cancel job
docker exec flink-jobmanager flink cancel JOB_ID

# Savepoint
docker exec flink-jobmanager flink savepoint JOB_ID /tmp/flink-savepoints
```