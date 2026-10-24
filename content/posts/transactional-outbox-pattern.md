+++
date = '2026-10-24'
title = 'The Transactional Outbox Pattern'
description = "How to Update a Database and Publish an Event Without Losing Either"
series = ['system design']
tags = ['golang', 'system-design']
+++

Say you're building the order service for an online store. When a customer checks out, two things need to happen: the order gets saved to the database, and an `OrderCreated` event gets published so the inventory service can reserve stock.

Simple enough, until you write the actual code:

```python
def create_order(customer_id, items):
    order_id = save_order(customer_id, items)
    publish_event("OrderCreated", {"order_id": order_id, "items": items})
    return order_id
```

Two operations, two different systems: your database and the message broker. And that's the problem. Nothing guarantees both happen together.

## Where it breaks

The database write can succeed and the publish can fail. Broker's down, network blips, whatever. The order exists, but inventory never hears about it. Nobody reserves stock. The customer thinks they bought something that might not ship.

Flip it around: the publish succeeds but the database write fails, or the transaction rolls back afterward. Now inventory is reserving stock for an order that doesn't exist.

Your first instinct might be to wrap both calls in a transaction:

```python
db.begin_transaction()
order_id = save_order(customer_id, items)
publish_event("OrderCreated", {...})   # still outside the DB
db.commit_transaction()
```

This doesn't help. A database transaction only controls the database. It has no idea the broker exists. If the commit fails after the event was already published, you can't call the broker and ask for the message back — it's already in flight, maybe already consumed.

You could reach for two-phase commit across the database and the broker, but most brokers don't support it, and even the ones that do tie your service tightly to both systems — a steep price for an atomicity guarantee you'll rarely need that badly.

## The fix: write the event to the database

The trick is to stop treating "save data" and "send event" as two operations against two systems. Instead, write the event into the *same* database, in the *same* transaction as the order:

```sql
CREATE TABLE outbox (
    event_id UUID PRIMARY KEY,
    aggregate_type VARCHAR(100),
    aggregate_id VARCHAR(100),
    event_type VARCHAR(100),
    payload JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    published BOOLEAN DEFAULT FALSE
);
```

```python
def create_order(customer_id, items):
    order_id = str(uuid4())
    event_id = str(uuid4())

    db.begin_transaction()
    db.execute(
        "INSERT INTO orders (order_id, customer_id, status) VALUES (%s, %s, %s)",
        order_id, customer_id, "PENDING",
    )
    db.execute(
        """INSERT INTO outbox (event_id, aggregate_type, aggregate_id, event_type, payload)
           VALUES (%s, %s, %s, %s, %s)""",
        event_id, "Order", order_id, "OrderCreated",
        json.dumps({"order_id": order_id, "customer_id": customer_id, "items": items}),
    )
    db.commit_transaction()
    return order_id
```

The order row and the outbox row live or die together. There's no window where one exists without the other, because a single `COMMIT` covers both. You've turned a "database write + network call" problem into a plain old database write, which your database already knows how to do atomically.

Notice the order service never talks to the broker directly anymore. It just writes rows.

## Getting the event out of the outbox

Writing the event safely is only half the job. Something still needs to move rows from the `outbox` table to the message broker. That something is usually called a **message relay**.

The simplest version is a poller: a background process that periodically checks the outbox for unpublished rows, sends them to the broker, then marks them published.

```python
def relay_loop():
    while True:
        rows = db.execute(
            "SELECT * FROM outbox WHERE published = FALSE ORDER BY created_at LIMIT 100"
        )
        for row in rows:
            broker.publish(row.event_type, row.payload)
            db.execute(
                "UPDATE outbox SET published = TRUE WHERE event_id = %s", row.event_id
            )
        time.sleep(1)
```

This works, and it's easy to reason about. Its downside is latency — events sit in the table until the next poll — and the constant `SELECT ... WHERE published = FALSE` adds load as the table grows.

The more production-grade approach is **Change Data Capture (CDC)**: instead of polling the table, you tail the database's transaction log (Postgres's WAL, MySQL's binlog) and stream inserts out as they happen. Debezium is the tool most people reach for here — it's free, built on Kafka, and supports Postgres, MySQL, MongoDB, and SQL Server out of the box. With CDC, there's no polling query and closer to real-time delivery, at the cost of a bit more infrastructure to run.

Either way, the relay is a separate concern from your application code. The order service's only job is to write rows in a transaction; getting those rows to the broker is someone else's problem, literally a different process.

## What you don't get for free

The outbox pattern guarantees that an event gets created if and only if the transaction that created it committed. That's it — and it's worth being precise about what it doesn't guarantee.

It doesn't guarantee the event reaches the broker exactly once. The relay can publish an event and then crash before marking the outbox row as sent. On restart, it sees an unpublished row and sends it again. So consumers can receive the same `OrderCreated` event twice.

And it doesn't guarantee delivery to consumers at all — the outbox only guarantees the event was safely *created*. Getting it delivered is a separate mechanism: the relay, plus whatever guarantees your broker gives you.

And a word of caution: it's tempting to let other services just read your `outbox` table directly instead of going through the broker. Don't. That turns your internal table into a de facto API, and now every consumer is coupled to your schema. Keep the outbox as an internal implementation detail; the broker is the actual interface.

## Why this is worth the extra table

Compare the two failure modes:

Without the outbox: the order commits, the publish fails, and the event is gone forever. No retry can recover it because your application has already moved on.

With the outbox: the order commits, the outbox row commits with it, and even if the relay crashes ten times in a row, the row is sitting right there waiting to be published on the next attempt. You've traded a *permanent* loss for, at worst, a *duplicate* delivery.

Duplicates are a much easier problem than lost events. A lost event needs someone to notice something never happened and manually replay it. A duplicate event just needs the consumer to recognize it's seen this one before and skip it — which is exactly what the inbox pattern, next in this series, is for.

**One line to remember**: outbox trades "did this even happen" for "did this happen twice," and the second problem is solvable in code.
