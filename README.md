# 🚀 Change Data Capture (CDC) with Kafka, Debezium, PostgreSQL, and ClickHouse using Docker Compose

This repository contains a Docker Compose configuration to set up a complete CDC (Change Data Capture) platform using Kafka ecosystem components, Debezium for CDC streaming, PostgreSQL as a source database, and ClickHouse as a destination database. The setup also includes Schema Registry, Kafka REST Proxy, and a web console for monitoring.

---

## 📦 Services Overview

| Service             | Hostname             | Ports                     | Description                                  |
|---------------------|----------------------|---------------------------|----------------------------------------------|
| `cdc-zookeeper`     | cdc-zookeeper        | 3181:2181                 | Zookeeper for Kafka cluster coordination     |
| `cdc-broker`        | cdc-broker           | 29092, 9092, 9101, 9094   | Kafka broker                                 |
| `cdc-debezium`      | cdc-debezium         | 9083                      | Debezium Connect service for CDC connectors |
| `cdc-schema-registry` | cdc-schema-registry | 9081                      | Kafka Schema Registry for Avro schema storage|
| `cdc-rest-proxy`    | cdc-rest-proxy       | 9082                      | Kafka REST Proxy to interact via HTTP       |
| `cdc-redpanda`      | cdc-redpanda         | 9080                      | Redpanda Console for monitoring Kafka topics|
| `cdc-postgres`      | cdc-postgres         | 5436                      | PostgreSQL database configured for logical replication |
| `cdc-clickhouse`    | cdc-clickhouse       | 8123, 9004                | ClickHouse analytical database               |

---

## 🗂️ Folder Structure

```text
.
├── docker-compose.yml        # Docker Compose configuration file
├── kafka/
│   ├── connectors/           # Debezium connector configurations
│   ├── plugins/              # Kafka Connect plugins
│   └── kafka-data/           # Persistent Kafka broker data
├── postgres/
│   └── postgres_data/        # Persistent PostgreSQL data volume
├── clickhouse/
│   ├── clickhouse-server/
│   │   ├── config.d/
│   │   │   └── config.xml    # ClickHouse server configs
│   │   └── users.d/
│   │       └── users.xml     # ClickHouse user configs
│   └── data/                 # ClickHouse persistent data
└── Dockerfile_debezium       # Custom Dockerfile for Debezium image build

```

## 🚀 Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/your-username/your-repo.git
cd your-repo
```

### 2. Build the containers

```bash
docker-compose up -d --build
```

This will:
- Build and start Zookeeper, Kafka broker, Debezium connect, Schema Registry, Kafka REST Proxy, Redpanda Console, PostgreSQL, and ClickHouse containers.
- Setup network with static IPs for stable inter-container communication.
- Enable healthchecks to ensure service readiness before dependent services start.

### 3. Verify Services
Check the health of your services:

```bash
docker ps
docker-compose logs -f
```


## 💡 Example Use Case

This guide shows a simple usage flow for capturing change data from a `users` table in PostgreSQL using different Debezium connector configurations.

### 🧱 Step 1: Create the `users` table

Run the following SQL to create a `users` table in your PostgreSQL database:

```sql
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(255) NOT NULL,
    account_type VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 🚀 Step 2: Start Debezium Connectors

Start the connectors by running the following commands in your terminal (you can choose the connectors in folder `connectors` or use your own connector.):

```bash
curl --location --request PUT 'http://localhost:9083/connectors/after-state-only-json/config' \
  --header 'Content-Type: application/json' \
  --data-binary @./connectors/after-state-only-json.json
```
Make sure Debezium, Kafka, and your PostgreSQL container are already running before executing these commands.

### 📝 Step 3: Insert Sample Data into the Table

Use the following SQL to insert sample records into the users table. These changes will be captured by the configured Debezium connectors.

```sql
INSERT INTO public.users (user_id, username, account_type, created_at, updated_at) 
VALUES (1, 'test_user1', 'user', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);

INSERT INTO public.users (user_id, username, account_type, created_at, updated_at) 
VALUES (2, 'test_user2', 'user', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP),
       (3, 'test_user3', 'user', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);

INSERT INTO public.users (user_id, username, account_type, created_at, updated_at) 
VALUES (4, 'test_user4', 'user', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP),
       (5, 'test_user5', 'user', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);
```


### 🔍 Step 4: Verify the Topics in Kafka UI

Once your connectors are running and you've inserted some data into the `users` table, you can verify that the changes are being captured by Debezium and published to Kafka topics.

1. Open your browser and go to:
2. This opens the Kafka UI where you can:
    - Browse all available topics
    - Check messages inside topics (e.g. `after-state-only-json.public.users`)
    - Inspect the event payloads in JSON or AVRO format depending on the connector configuration

This step is useful to validate that your setup is working correctly and to see the structure of captured change events.

### Step 5: Create Destination Table in ClickHouse

```sql
CREATE TABLE cdc_users_after_state_only
(
    user_id       UInt32,
    username      String,
    account_type  String,
    created_at    DateTime,
    updated_at    DateTime
)
ENGINE = ReplacingMergeTree(updated_at)
ORDER BY user_id;
```

### Step 6: Create Kafka Engine Table in ClickHouse to Read Kafka Topic

```sql
CREATE TABLE cdc.kafka_users_after_state_only_json
(
    user_id       UInt32,
    username      String,
    account_type  String,
    created_at    DateTime,
    updated_at    DateTime
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'cdc-broker:29092',
    kafka_topic_list = 'after-state-only-json.public.users',
    kafka_group_name = 'test',
    kafka_format = 'JSONEachRow',
    kafka_num_consumers = 1,
    kafka_skip_broken_messages = 1;
```

### Step 7: Check the Kafka Engine Table Data

```sql
SET stream_like_engine_allow_direct_select = 1;
SELECT * FROM kafka_users_after_state_only_json;
```

### Step 8: Create Materialized View to Populate Destination Table

```sql
CREATE MATERIALIZED VIEW cdc_users_mv TO cdc_users_after_state_only AS
SELECT
    user_id,
    username,
    account_type,
    created_at,
    updated_at,
    1 AS version
FROM kafka_users_after_state_only_json;
```

### Step 9: Verify Data in the Destination Table

```sql
SELECT * FROM cdc_users_after_state_only;
```


## 🔧 Service Details & Usage

### Kafka & Zookeeper
- **Zookeeper** runs on port **3181** (mapped internally to **2181**).
- **Kafka Broker** is accessible on ports:
  - **29092** (internal Kafka listener)
  - **9092** (external)
- Kafka topics can be managed via **REST Proxy** on port **9082**.


### Debezium Connect (CDC)
- Debezium Connect REST API endpoint:  
  `http://localhost:9083/connectors`
- Use this to add connectors for capturing change events from PostgreSQL.


### Schema Registry
- Schema Registry UI/API available at:  
  `http://localhost:9081`
- Manages Avro schemas for Kafka topics.


### Redpanda Console
- Web UI for Kafka monitoring:  
  `http://localhost:9080`
- Allows topic browsing, message inspection, and cluster overview.


### PostgreSQL (CDC Source)
- Accessible at: `localhost:5436`
- Credentials:  
  - User: `postgres`  
  - Password: `postgres`
- Configured with logical replication (`wal_level=logical`) for CDC.


### ClickHouse (CDC Destination)
- HTTP interface on port **8123**.
- TCP interface on port **9004**.


## ⚙️ Configuration Notes
- Kafka data persists under: `./kafka/kafka-data`
- PostgreSQL data persists under: `./postgres/postgres_data`
- ClickHouse data and configs persist under: `./clickhouse`
- Docker network `network-docker_default` is external; ensure it exists or adjust accordingly.


## ❗ Troubleshooting: Error Building Broker

If you encounter an error while building Broker:

Delete all data in the Kafka data directory:

```bash
rm -rf kafka/kafka-data/*
```




