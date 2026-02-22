# pg-cdc.json --- Debezium PostgreSQL Source Connector (Explained)

This file configures Debezium to capture changes from PostgreSQL and
publish clean events to Kafka.

------------------------------------------------------------------------

## Connector Basics

**connector.class**\
Type of connector.\
Here we use the Debezium PostgreSQL CDC connector.

**database.hostname / port / user / password / dbname**\
Connection details used by Debezium to connect to the database.

------------------------------------------------------------------------

## Kafka Topic Settings

**topic.prefix = pg**\
All captured tables will be published under topics starting with:

`pg.<schema>.<table>`

Example: `pg.public.customers`

------------------------------------------------------------------------

## Replication Configuration

**slot.name = debezium**\
Logical replication slot inside PostgreSQL.\
This allows Debezium to continuously read WAL changes.

**publication.autocreate.mode = filtered**\
Debezium automatically creates a publication containing only the
selected tables.

**plugin.name = pgoutput**\
Postgres logical decoding plugin used to stream changes.

------------------------------------------------------------------------

## Table Filtering

**table.include.list = public.customers**\
Only this table will produce events.

You can add multiple tables: `public.customers,public.orders`

------------------------------------------------------------------------

## Delete Handling

**tombstones.on.delete = false**\
Prevents Kafka tombstone messages after delete operations. Keeps webhook
payload clean.

------------------------------------------------------------------------

## Transforms --- Making Clean Events

Debezium normally sends large technical events:

    before / after / source / lsn / txid / timestamps

We convert them into simple business JSON.

### unwrap transform

Extracts only the row data.

-   drop.tombstones = true → removes Kafka tombstone records
-   delete.handling.mode = rewrite → sends minimal delete event
-   add.fields = op → keeps operation type (c/u/d)

Result:

    {"id":1,"name":"Anna","email":"anna@test.com","__op":"u"}

------------------------------------------------------------------------

### rename transform

Renames Debezium operation field:

    __op → event

Final result:

    {"id":1,"name":"Anna","email":"anna@test.com","event":"u"}

------------------------------------------------------------------------

## Event Types

  event   Meaning
  ------- ---------
  c       insert
  u       update
  d       delete

------------------------------------------------------------------------

## Final Output

Your Kafka + Webhook now receives small, clean business events instead
of database logs:

    {"id":2,"name":"John","email":"john@test.com","event":"c"}
    {"id":2,"name":"Johnny","email":"john@test.com","event":"u"}
    {"id":2,"event":"d"}
