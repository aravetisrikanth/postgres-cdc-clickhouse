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
```bash
# Deploy Zookeeper
kubectl apply -f k8s/zookeeper.yaml
kubectl wait --for=condition=ready pod -l app=zookeeper --timeout=120s
# Deploy Kafka
kubectl apply -f k8s/kafka.yaml
kubectl wait --for=condition=ready pod -l app=kafka-broker --timeout=120s
# Deploy Schema Registry
kubectl apply -f k8s/schema-registry.yaml
# Deploy Apicurio Registry
kubectl apply -f k8s/apicurio-registry.yaml
# Deploy PostgreSQL
kubectl apply -f k8s/postgres.yaml
kubectl wait --for=condition=ready pod -l app=postgres --timeout=120s
# Deploy Debezium
kubectl apply -f k8s/debezium.yaml
kubectl wait --for=condition=ready pod -l app=debezium --timeout=120s

```

2. **Deploy ClickHouse Components**

```bash
# Create ConfigMaps for ClickHouse (you'll need to create these config files first)
kubectl create configmap clickhouse-config --from-file=config.xml
kubectl create configmap clickhouse-users --from-file=users.xml

# Deploy ClickHouse Keeper
kubectl apply -f k8s/clickhouse-keeper.yaml
kubectl wait --for=condition=ready pod -l app=clickhouse-keeper --timeout=120s

# Deploy ClickHouse
kubectl apply -f k8s/clickhouse.yaml
kubectl wait --for=condition=ready pod -l app=clickhouse --timeout=120s

# Check if Keeper is running
kubectl get pods -l app=clickhouse-keeper

# Check Keeper logs
kubectl logs -l app=clickhouse-keeper

# Check if ClickHouse can connect to Keeper
kubectl exec -it $(kubectl get pod -l app=clickhouse -o jsonpath='{.items[0].metadata.name}') -- clickhouse-client --query="SELECT * FROM system.zookeeper WHERE path = '/'"

```

3. **Deploy Monitoring**

```bash
# Deploy Redpanda Console
kubectl apply -f k8s/redpanda-console.yaml

# Wait for the pod to be ready
kubectl wait --for=condition=ready pod -l app=redpanda-console --timeout=120s

# Check the status
kubectl get pods -l app=redpanda-console

# Port forward to access the UI locally
kubectl port-forward svc/redpanda-console 8080:8080

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

# or you can do port forward to the postgres pod so you can connect to it locally using pgadmin, pgcli, psql etc.
kubectl port-forward svc/postgres 5432:5432

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
('user1', 'Employee'),
('user2', 'Contractor'),
('user3', 'Vendor');

INSERT INTO users (username, account_type) VALUES
('user4', 'Guest')

INSERT INTO users (username, account_type) VALUES
(
'user5', 'Visitor'
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
# Connect to Clickhouse
kubectl exec -it $(kubectl get pod -l app=clickhouse -o jsonpath='{.items[0].metadata.name}') bash 

# or you can do port forward to the clickhouse pod so you can connect to it locally using clickhouse-client, DbVisualizer etc.
kubectl port-forward svc/clickhouse 8123:8123
```
# Helper queries to create the clickhouse mv, tables
```sql
DROP DATABASE IF NOT EXISTS shop_db

CREATE DATABASE IF NOT EXISTS shop_db;

-- Create the target table
CREATE TABLE IF NOT EXISTS shop_db.users
(
    user_id UInt32,
    username String,
    account_type String,
    updated_at DateTime64(6),
    created_at DateTime64(6),
    is_deleted UInt8,
    _version UInt64
) ENGINE = ReplacingMergeTree(_version)
ORDER BY (user_id);

-- Create the Kafka source table
CREATE TABLE IF NOT EXISTS shop_db.users_kafka
(
    payload Tuple(
        user_id UInt32,
        username String,
        account_type String,
        updated_at Int64,
        created_at Int64,
        __deleted String
    )
) ENGINE = Kafka('kafka-broker:29092', 'shop_db.public.users', 'clickhouse_consumer_group', 'JSONEachRow');

-- Create materialized view
CREATE MATERIALIZED VIEW IF NOT EXISTS shop_db.users_mv
TO shop_db.users
AS SELECT
    payload.1 as user_id,
    payload.2 as username,
    payload.3 as account_type,
    fromUnixTimestamp64Micro(payload.4) as updated_at,
    fromUnixTimestamp64Micro(payload.5) as created_at,
    payload.6 = 'true' as is_deleted,
    toUInt64(payload.4) as _version
FROM shop_db.users_kafka
WHERE length(_raw_message) > 0;

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