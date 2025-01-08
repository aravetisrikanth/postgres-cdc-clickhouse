# Kubernetes CDC Pipeline with Debezium

This project sets up a Change Data Capture (CDC) pipeline using Debezium, Kafka, PostgreSQL, and ClickHouse in Kubernetes.

## Architecture Components

- PostgreSQL: Source database
- Debezium: CDC connector
- Apache Kafka: Message broker
- ClickHouse: Target database
- Redpanda Console: Kafka UI for monitoring
- ClickHouse Keeper: Coordination service for ClickHouse

## Prerequisites

- Kubernetes cluster
- kubectl configured
- Access to container registries

## Directory Structure
├── k8s/
│ ├── zookeeper.yaml
│ ├── kafka.yaml
│ ├── schema-registry.yaml
│ ├── postgres.yaml
│ ├── debezium.yaml
│ ├── clickhouse.yaml
│ ├── clickhouse-keeper.yaml
│ ├── redpanda-console.yaml
│ ├── postgres-source-connector.yaml
│ └── create-connector-job.yaml
└── README.md
- k8s/ - Kubernetes manifests
- postgres-source-connector.yaml - Debezium PostgreSQL source connector configuration
- clickhouse-target-connector.yaml - Debezium ClickHouse target connector configuration

## Installation Steps

1. **Deploy Core Services**
Deploy Zookeeper
kubectl apply -f k8s/zookeeper.yaml
kubectl wait --for=condition=ready pod -l app=zookeeper --timeout=120s
Deploy Kafka
kubectl apply -f k8s/kafka.yaml
kubectl wait --for=condition=ready pod -l app=kafka-broker --timeout=120s
Deploy Schema Registry
kubectl apply -f k8s/schema-registry.yaml
Deploy Apicurio Registry
kubectl apply -f k8s/apicurio-registry.yaml
Deploy PostgreSQL
kubectl apply -f k8s/postgres.yaml
kubectl wait --for=condition=ready pod -l app=postgres --timeout=120s
Deploy Debezium
kubectl apply -f k8s/debezium.yaml
kubectl wait --for=condition=ready pod -l app=debezium --timeout=120s

2. **Deploy ClickHouse Components**

```bash
# Deploy ClickHouse Keeper
kubectl apply -f k8s/clickhouse-keeper.yaml
kubectl wait --for=condition=ready pod -l app=clickhouse-keeper --timeout=120s

# Deploy ClickHouse
kubectl apply -f k8s/clickhouse.yaml
kubectl wait --for=condition=ready pod -l app=clickhouse --timeout=120s
```

3. **Deploy Monitoring**

```bash
# Deploy Redpanda Console
kubectl apply -f k8s/redpanda-console.yaml
```

4. **Configure CDC**

```bash
# Deploy connector configuration
kubectl apply -f k8s/postgres-source-connector.yaml

# Create the connector
kubectl apply -f k8s/create-connector-job.yaml
```

## Verification Steps

1. **Check All Pods**
```bash
kubectl get pods
```

2. **Verify PostgreSQL**
```bash
# Connect to PostgreSQL
kubectl exec -it $(kubectl get pod -l app=postgres -o jsonpath='{.items[0].metadata.name}') -- psql -U postgres -d shop_db

# Create a test table
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    account_type VARCHAR(20) NOT NULL,
    updated_at TIMESTAMP DEFAULT timezone('UTC', CURRENT_TIMESTAMP),
    created_at TIMESTAMP DEFAULT timezone('UTC', CURRENT_TIMESTAMP)
);

# Insert test data
INSERT INTO users (username, account_type) VALUES
('user1', 'John'),
('user2', 'Jack'),
('user3', 'James');

INSERT INTO users (username, account_type) VALUES
('user4', 'Json')

INSERT INTO users (username, account_type) VALUES
(
'user5', 'Jared'
)
```

3. **Verify Kafka Topics**
```bash
# List topics
kubectl exec -it $(kubectl get pod -l app=kafka-broker -o jsonpath='{.items[0].metadata.name}') -- \
  kafka-topics --list --bootstrap-server localhost:9092

# Consume messages
kubectl exec -it $(kubectl get pod -l app=kafka-broker -o jsonpath='{.items[0].metadata.name}') -- \
  kafka-console-consumer --bootstrap-server kafka-broker:29092 \
  --topic postgres.public.test_table --from-beginning
```

4. **Check Debezium Connector Status**
```bash
kubectl exec -it $(kubectl get pod -l app=debezium -o jsonpath='{.items[0].metadata.name}') -- \
  curl -s http://localhost:8083/connectors/postgres-source-connector/status
```

# Create Clickhouse table
```bash
kubectl exec -it $(kubectl get pod -l app=clickhouse -o jsonpath='{.items[0].metadata.name}') -- \
  clickhouse-client --query "CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username String,
    account_type String,
    updated_at DateTime,
    created_at DateTime,
    kafka_time Nullable(DateTime),
    kafka_offset UInt64
) ENGINE = MergeTree() ORDER BY user_id, updated_at"
```
# Helper queries to create the clickhouse mv, tables
```sql
CREATE DATABASE shop;
CREATE TABLE shop.users
(    
  user_id UInt32,    
  username String,    
  account_type String,    
  updated_at DateTime,    
  created_at DateTime,    
  kafka_time Nullable(DateTime),
  kafka_offset UInt64
  )
  ENGINE = ReplacingMergeTree
  ORDER BY (user_id, updated_at)
  SETTINGS index_granularity = 8192;
  
  CREATE TABLE shop.kafka__users
  (    
    user_id UInt32,    
    username String,    
    account_type String,    
    updated_at UInt64,    
    created_at UInt64
  )
  ENGINE = Kafka
  SETTINGS kafka_broker_list = 'kafka-broker:29092',
  kafka_topic_list = 'shop.public.users',
  kafka_group_name = 'clickhouse',
  kafka_format = 'JSONEachRow', --??
  kafka_row_delimiter = '\n',
  kafka_schema = '{"type":"struct","fields":[{"field":"user_id","type":"int32"},{"field":"username","type":"string"},{"field":"account_type","type":"string"},{"field":"updated_at","type":"int64"},{"field":"created_at","type":"int64"}]}'

  CREATE MATERIALIZED VIEW kafka_shop_db.consumer__users TO shop.users
(
    user_id UInt32,
    username String,
    account_type String,
    updated_at DateTime,
    created_at DateTime,
    kafka_time Nullable(DateTime),
    kafka_offset UInt64
) AS
SELECT
    user_id,
    username,
    account_type,
    toDateTime(updated_at / 1000000) AS updated_at,
    toDateTime(created_at / 1000000) AS created_at,
    _timestamp AS kafka_time,
    _offset AS kafka_offset
FROM kafka_shop_db.kafka__users;

```

## Accessing UIs

1. **Redpanda Console**
```bash
kubectl port-forward svc/redpanda-console 8080:8080
# Access at http://localhost:8080
```

2. **ClickHouse**
```bash
kubectl port-forward svc/clickhouse 8123:8123
# Access at http://localhost:8123
```

## Troubleshooting

1. **Check Logs**
```bash
# Debezium logs
kubectl logs -l app=debezium

# Kafka logs
kubectl logs -l app=kafka-broker

# PostgreSQL logs
kubectl logs -l app=postgres
```

2. **Check Events**
```bash
kubectl get events --sort-by='.lastTimestamp'
```

3. **Common Issues**
- If connector fails to start, check Debezium logs
- Verify PostgreSQL is configured with `wal_level=logical`
- Ensure all services are running before creating the connector
- Check network connectivity between services

## Cleanup

```bash
# Delete all resources
kubectl delete -f k8s/
```

## Configuration Details

### Debezium Connector
The connector is configured to:
- Use JSON serialization
- Disable schema inclusion in messages
- Transform messages to extract new record state
- Monitor all tables in the public schema

### PostgreSQL
- Configured with logical replication enabled
- Default credentials: postgres/postgres
- Database name: shop_db

### Kafka
- Internal communication port: 29092
- External access port: 9092
- Single broker setup (for development)

## Production Considerations

1. **Security**
- Implement proper authentication and authorization
- Use Secrets for sensitive data
- Configure TLS for communication

2. **Storage**
- Add PersistentVolumes for stateful components
- Configure proper storage classes

3. **High Availability**
- Increase replicas for critical components
- Use StatefulSets for stateful services
- Configure proper anti-affinity rules

4. **Monitoring**
- Add Prometheus metrics
- Configure proper resource limits and requests
- Set up alerting

## Contributing

1. Fork the repository
2. Create your feature branch
3. Commit your changes
4. Push to the branch
5. Create a new Pull Request