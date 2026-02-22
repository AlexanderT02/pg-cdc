# http-sink.json --- HTTP Sink Connector (Explained)

This connector reads events from Kafka and sends them to an HTTP
endpoint (webhook).

------------------------------------------------------------------------

## Connector Basics

**connector.class**\
Uses the Confluent HTTP Sink Connector to send Kafka messages as HTTP
requests.

**tasks.max = 1**\
Number of parallel workers processing messages. For demos and ordering
guarantees we keep this at 1.

------------------------------------------------------------------------

## Kafka Source

**topics = pg.public.customers**\
The Kafka topic that contains Debezium events.

Every change in the database produces a new message here.

------------------------------------------------------------------------

## HTTP Destination

**http.api.url = http://webhook:8080**\
Target endpoint inside Docker network.

**request.method = POST**\
Each Kafka message becomes an HTTP POST request.

**headers = Content-Type:application/json**\
Send payload as JSON body.

------------------------------------------------------------------------

## Value Converter

**value.converter = StringConverter**

Kafka Connect internally stores structured records.\
The HTTP connector expects a raw JSON string.

This converter ensures the record is sent exactly as JSON text:

    {"id":2,"name":"Anna","email":"anna@test.com","event":"u"}

------------------------------------------------------------------------

## Internal Topics (Required by Confluent Connector)

The HTTP Sink uses internal Kafka topics for reliability tracking.

### bootstrap servers

Defines where Kafka is reachable inside Docker.

    kafka:29092

### success topic

Stores successful HTTP deliveries.

    http-sink-success

### error topic

Stores failed HTTP deliveries.

    http-sink-error

You can inspect failures:

    docker exec -it cdc-pg-kafka-1 kafka-console-consumer  --bootstrap-server kafka:29092  --topic http-sink-error  --from-beginning

------------------------------------------------------------------------

## Result

Each database change triggers:

Database change\
→ Kafka message\
→ HTTP POST

Example webhook request:

    POST / HTTP/1.1
    Content-Type: application/json

Body:

    {"id":2,"name":"Anna","email":"anna@test.com","event":"u"}

------------------------------------------------------------------------

## Why This Is Useful

This turns your database into an event source for:

-   microservices communication
-   ERP integrations
-   shop synchronization
-   webhook driven workflows
