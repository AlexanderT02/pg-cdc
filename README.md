# CDC Demo

This project demonstrates a Change Data Capture (CDC) pipeline:

Postgres → Debezium → Kafka → Kafka Connect → HTTP Webhook

It allows to capture database changes in real time and send them as
clean JSON events to an HTTP endpoint.

------------------------------------------------------------------------

## Architecture

1.  **Postgres 16**

    -   Logical replication enabled
    -   WAL level = logical

2.  **Debezium PostgreSQL Connector**

    -   Reads WAL changes
    -   Publishes events to Kafka

3.  **Kafka (Confluent Platform 7.5.0)**

4.  **Kafka Connect**

    -   Applies unwrap transform
    -   Sends events to HTTP webhook

5.  **HTTP Echo Server**

    -   Receives POST requests
    -   Displays JSON payload

------------------------------------------------------------------------

## Services (Docker Compose)

-   postgres (logical replication enabled)
-   zookeeper
-   kafka
-   connect
-   webhook (mendhak/http-https-echo)

------------------------------------------------------------------------

## How It Works

1.  Insert/Update/Delete in Postgres

2.  Debezium reads WAL changes

3.  Kafka receives event in topic:

    pg.public.customers

4.  Transform (unwrap SMT) converts event into clean JSON:

``` json
{"id": 2, "name": "Anna", "email": "anna@test.com", "event": "u"}
```

5.  HTTP Sink sends POST request to:

```{=html}
    http://webhook:8080
```


------------------------------------------------------------------------

## Start the Environment

``` bash
docker compose up -d --build
```

Wait \~30--60 seconds for Kafka and Connect to fully initialize.

------------------------------------------------------------------------

## Create Test Table

``` bash
docker exec -it cdc-pg-postgres-1 psql -U postgres -d app
```

``` sql
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name TEXT,
    email TEXT
);
```

------------------------------------------------------------------------

## Register Connectors

### Debezium Source

``` bash
curl -X POST http://localhost:8083/connectors   -H "Content-Type: application/json"   --data @connectors/pg-cdc.json
```

### HTTP Sink

``` bash
curl -X POST http://localhost:8083/connectors   -H "Content-Type: application/json"   --data @connectors/http-sink.json
```

------------------------------------------------------------------------

## Test CDC

``` sql
INSERT INTO customers (name,email) VALUES ('Test','test@test.com');
UPDATE customers SET name='Updated' WHERE id=1;
DELETE FROM customers WHERE id=1;
```

------------------------------------------------------------------------

## View Webhook Output

Do NOT rely only on browser refresh.

Instead, check container logs:

``` bash
docker logs -f cdc-pg-webhook-1
```

You will see:

``` http
POST / HTTP/1.1
```

With body:

``` json
{"id":1,"name":"Updated","email":"test@test.com","event":"u"}
```

------------------------------------------------------------------------

## Event Format

After the unwrap transform, the database operation is mapped to an event code:

| DB Action | Event |
|--------|------|
| INSERT | `"event": "c"` |
| UPDATE | `"event": "u"` |
| DELETE | `"event": "d"` |

### Example

```json
{
  "id": 4,
  "name": "New",
  "email": "new@test.com",
  "event": "c"
}
```

------------------------------------------------------------------------

## Troubleshooting

### Topic has data but no webhook calls?

Check sink status:

``` bash
curl http://localhost:8083/connectors/http-sink/status
```

### Verify Kafka topic:

``` bash
docker exec -it cdc-pg-kafka-1 kafka-console-consumer  --bootstrap-server kafka:29092  --topic pg.public.customers  --from-beginning
```

### Check Webhook logs:

``` bash
docker logs -f cdc-pg-webhook-1
```




