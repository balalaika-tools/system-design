# Message Queues and Event-Driven Architecture

> **Goal**: Understand how asynchronous messaging enables scalable, resilient, and decoupled systems.

---

## 1. Why Message Queues?

### The Synchronous Problem

```
┌─────────────────────────────────────────────────────────────────────┐
│  SYNCHRONOUS FLOW                                                   │
│                                                                     │
│  User → API → Process Payment → Send Email → Update Analytics       │
│                                                                     │
│  Problems:                                                          │
│  • User waits for everything (slow)                                 │
│  • Email service down = entire request fails                        │
│  • Can't scale email sending independently                          │
│  • Retry logic becomes complex                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Asynchronous Solution

```
┌─────────────────────────────────────────────────────────────────────┐
│  ASYNCHRONOUS FLOW                                                  │
│                                                                     │
│  User → API → Process Payment → Return Success                      │
│                    ↓                                                │
│               Queue Message                                         │
│                    ↓                                                │
│         ┌─────────────────────┐                                     │
│         ↓                     ↓                                     │
│    Email Worker        Analytics Worker                             │
│                                                                     │
│  Benefits:                                                          │
│  • User gets fast response                                          │
│  • Components fail independently                                    │
│  • Scale workers based on queue depth                               │
│  • Built-in retry with message redelivery                           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Core Concepts

### Message

```json
{
  "id": "msg-123",
  "type": "order.created",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "order_id": "order-456",
    "user_id": "user-789",
    "total": 99.99
  }
}
```

### Queue

A **queue** holds messages until consumers process them.

```
Producer → [msg3][msg2][msg1] → Consumer
           └─────Queue─────┘
```

**Key properties:**
- **FIFO** (usually): First in, first out
- **Durable**: Survives restarts
- **Acknowledged**: Messages removed only after processing confirmed

### Topic (Pub/Sub)

A **topic** broadcasts messages to multiple subscribers.

```
Producer → Topic → Consumer A
                → Consumer B
                → Consumer C
```

Each subscriber gets every message (fan-out).

---

## 3. Queue vs Topic

| Aspect | Queue | Topic |
|--------|-------|-------|
| Delivery | One consumer per message | All subscribers get message |
| Use case | Task distribution | Event broadcasting |
| Pattern | Work queue | Publish/Subscribe |
| Example | Process orders | Notify systems of order |

### Combined Pattern

```
┌─────────────────────────────────────────────────────────────────────┐
│  Order Service publishes to "orders" topic                          │
│                                                                     │
│  orders ──→ email-queue ──→ Email Workers (3 instances)             │
│        ──→ analytics-queue ──→ Analytics Workers (2 instances)      │
│        ──→ inventory-queue ──→ Inventory Worker                     │
│                                                                     │
│  Each service has its own queue subscribed to the topic             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Message Queue Technologies

### RabbitMQ

```
Best for: Traditional messaging, complex routing
Protocol: AMQP
Strengths: Flexible routing, mature, many language clients
```

```python
import pika

# Producer
connection = pika.BlockingConnection(pika.ConnectionParameters('rabbitmq'))
channel = connection.channel()
channel.queue_declare(queue='tasks', durable=True)
channel.basic_publish(
    exchange='',
    routing_key='tasks',
    body='{"task": "send_email"}',
    properties=pika.BasicProperties(delivery_mode=2)  # Persistent
)

# Consumer
def callback(ch, method, properties, body):
    process_task(body)
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(queue='tasks', on_message_callback=callback)
channel.start_consuming()
```

### Apache Kafka

```
Best for: High-throughput streaming, event sourcing
Protocol: Custom binary
Strengths: Massive scale, replay, ordering within partition
```

```python
from kafka import KafkaProducer, KafkaConsumer

# Producer
producer = KafkaProducer(bootstrap_servers='kafka:9092')
producer.send('orders', value=b'{"order_id": "123"}')

# Consumer
consumer = KafkaConsumer('orders', bootstrap_servers='kafka:9092')
for message in consumer:
    process_order(message.value)
```

**Kafka Key Concepts:**

```
┌─────────────────────────────────────────────────┐
│  Topic: orders                                  │
│  ┌──────────────────────────────────────────┐   │
│  │ Partition 0: [msg1][msg2][msg3]          │   │
│  │ Partition 1: [msg4][msg5]                │   │
│  │ Partition 2: [msg6][msg7][msg8][msg9]    │   │
│  └──────────────────────────────────────────┘   │
│                                                 │
│  • Messages ordered WITHIN partition            │
│  • Consumer groups for parallel processing      │
│  • Offset tracking for replay                   │
└─────────────────────────────────────────────────┘
```

### Redis Streams / Pub/Sub

```
Best for: Simple use cases, already have Redis
Strengths: Fast, simple, no extra infrastructure
```

```python
import redis

r = redis.Redis()

# Publish
r.xadd('mystream', {'event': 'order_created', 'order_id': '123'})

# Consume
r.xread({'mystream': '0'}, block=5000)
```

### Cloud Managed

| Cloud | Queue Service | Streaming Service |
|-------|--------------|-------------------|
| AWS | SQS | Kinesis, MSK (Kafka) |
| GCP | Cloud Pub/Sub | Cloud Pub/Sub |
| Azure | Service Bus | Event Hubs |

---

## 5. Delivery Guarantees

### At-Most-Once

```
┌─────────────────────────────────────────────────┐
│  Fire and forget                                │
│                                                 │
│  • Message may be lost                          │
│  • No retries                                   │
│  • Fastest                                      │
│                                                 │
│  Use: Metrics, logs, non-critical events        │
└─────────────────────────────────────────────────┘
```

### At-Least-Once

```
┌─────────────────────────────────────────────────┐
│  Guaranteed delivery with possible duplicates   │
│                                                 │
│  • Message delivered at least once              │
│  • May be delivered multiple times              │
│  • Consumer must be idempotent                  │
│                                                 │
│  Use: Most business operations                  │
└─────────────────────────────────────────────────┘
```

### Exactly-Once (Hard!)

```
┌─────────────────────────────────────────────────┐
│  Each message processed exactly once            │
│                                                 │
│  • Requires coordination                        │
│  • Expensive/complex                            │
│  • Often "effectively once" via idempotency     │
│                                                 │
│  Use: Financial transactions, inventory         │
└─────────────────────────────────────────────────┘
```

> **Pro tip**: Design for at-least-once and make operations idempotent. Exactly-once is expensive.

---

## 6. Idempotency: The Key to Reliability

### The Problem

```
┌─────────────────────────────────────────────────┐
│  1. Consumer processes message                  │
│  2. Processing succeeds                         │
│  3. Ack fails (network issue)                   │
│  4. Message redelivered                         │
│  5. Consumer processes AGAIN                    │
│                                                 │
│  Result: Double-charged customer 💸   💸       │
└─────────────────────────────────────────────────┘
```

### The Solution: Idempotent Operations

```python
def process_payment(message):
    payment_id = message['payment_id']
    
    # Check if already processed
    if db.exists("processed_payments", payment_id):
        return  # Already done, skip
    
    # Process payment
    charge_customer(message['amount'])
    
    # Mark as processed
    db.insert("processed_payments", payment_id)
```

**Idempotency Keys:**
- Use message ID or business ID (order_id, payment_id)
- Store in database with unique constraint
- Check before processing

---

## 7. Common Patterns

### Pattern 1: Work Queue (Task Distribution)

```
┌─────────────────────────────────────────────────┐
│  Multiple workers share the load                │
│                                                 │
│  Producer → Queue → Worker 1                    │
│                  → Worker 2                     │
│                  → Worker 3                     │
│                                                 │
│  Each message processed by ONE worker           │
└─────────────────────────────────────────────────┘
```

**Use case**: Image processing, email sending, report generation

### Pattern 2: Pub/Sub (Event Broadcasting)

```
┌─────────────────────────────────────────────────┐
│  All subscribers receive every message          │
│                                                 │
│  Publisher → Topic → Email Service              │
│                   → Analytics Service           │
│                   → Notification Service        │
│                                                 │
│  Each service gets ALL messages                 │
└─────────────────────────────────────────────────┘
```

**Use case**: Order events, user activity, system events

### Pattern 3: Request-Reply

```
┌─────────────────────────────────────────────────┐
│  Async request with response                    │
│                                                 │
│  Client → Request Queue → Worker                │
│                              ↓                  │
│  Client ← Reply Queue ←──────┘                  │
│                                                 │
│  Correlation ID matches request to response     │
└─────────────────────────────────────────────────┘
```

**Use case**: Async API calls, distributed RPC

### Pattern 4: Dead Letter Queue

```
┌─────────────────────────────────────────────────┐
│  Failed messages go to separate queue           │
│                                                 │
│  Main Queue → Worker → Success                  │
│                     → Failure (retry)           │
│                     → Max retries → DLQ         │
│                                                 │
│  DLQ: inspect, fix, replay failed messages      │
└─────────────────────────────────────────────────┘
```

**Critical**: Always have a DLQ for production systems.

---

## 8. Event-Driven Architecture

### Events vs Commands

| Aspect | Event | Command |
|--------|-------|---------|
| Intent | "This happened" | "Do this" |
| Coupling | Loose | Tight |
| Naming | Past tense | Imperative |
| Example | `OrderCreated` | `SendEmail` |

```python
# Event: Something happened, react if you care
{
    "type": "order.created",
    "data": {"order_id": "123", "total": 99.99}
}

# Command: Direct instruction to do something
{
    "type": "send_email",
    "data": {"to": "user@example.com", "template": "order_confirmation"}
}
```

### Event Sourcing

```
┌─────────────────────────────────────────────────────────────────────┐
│  Instead of storing current state, store events                     │
│                                                                     │
│  Traditional: UPDATE accounts SET balance = 100 WHERE id = 1        │
│                                                                     │
│  Event Sourced:                                                     │
│  [AccountOpened(id=1, balance=0)]                                   │
│  [MoneyDeposited(id=1, amount=150)]                                 │
│  [MoneyWithdrawn(id=1, amount=50)]                                  │
│                                                                     │
│  Current balance = replay events = 0 + 150 - 50 = 100               │
└─────────────────────────────────────────────────────────────────────┘
```

**Benefits:**
- Complete audit trail
- Replay events for debugging
- Rebuild state from events
- Time travel queries

**Challenges:**
- Event schema evolution
- Eventual consistency
- Query complexity

---

## 9. Handling Failures

### Consumer Failure

```python
def consume_message(message):
    try:
        process(message)
        ack(message)  # Remove from queue
    except TransientError:
        nack(message)  # Retry later
    except PermanentError:
        send_to_dlq(message)  # Give up, investigate
```

### Poison Messages

```
┌─────────────────────────────────────────────────┐
│  Message that always fails                      │
│                                                 │
│  1. Message processed                           │
│  2. Error, retry                                │
│  3. Error, retry                                │
│  4. Error, retry (blocking queue!)              │
│                                                 │
│  Solution: Max retry count → Dead Letter Queue  │
└─────────────────────────────────────────────────┘
```

### Ordering Challenges

```
┌──────────────────────────────────────────────────┐
│  Events may arrive out of order                  │
│                                                  │
│  User Created (timestamp: 10:00)                 │
│  User Updated (timestamp: 10:01) ← arrives first │
│  User Created (timestamp: 10:00) ← arrives second│
│                                                  │
│  Solution: Version/timestamp checking            │
│  Solution: Partition by user_id (Kafka)          │
└──────────────────────────────────────────────────┘
```

---

## 10. Scaling Consumers

### Competing Consumers

```
┌─────────────────────────────────────────────────┐
│  Queue depth increasing?                        │
│       ↓                                         │
│  Add more consumers                             │
│                                                 │
│  [msg][msg][msg][msg][msg] → Consumer 1         │
│                           → Consumer 2          │
│                           → Consumer 3          │
└─────────────────────────────────────────────────┘
```

### Consumer Groups (Kafka)

```
┌─────────────────────────────────────────────────┐
│  Partitions distributed among group members     │
│                                                 │
│  Topic (3 partitions)                           │
│  Consumer Group: "order-processors"             │
│                                                 │
│  Partition 0 → Consumer A                       │
│  Partition 1 → Consumer B                       │
│  Partition 2 → Consumer C                       │
│                                                 │
│  Max parallelism = number of partitions         │
└─────────────────────────────────────────────────┘
```

### Backpressure

```python
# Don't fetch more than you can process
channel.basic_qos(prefetch_count=10)

# Consumer only gets 10 unacked messages at a time
```

---

## 11. When to Use Message Queues

### Good Use Cases

| Scenario | Why Queue Helps |
|----------|-----------------|
| Email/SMS sending | Async, retry, rate limiting |
| Image/video processing | CPU intensive, scale independently |
| Analytics/logging | Fire and forget, high volume |
| Order processing | Reliability, retry, audit |
| Microservice communication | Decoupling, resilience |
| Scheduled tasks | Delayed delivery |

### When NOT to Use

| Scenario | Better Alternative |
|----------|-------------------|
| Real-time responses needed | Synchronous API |
| Simple CRUD | Direct database |
| Strong consistency required | Transactions |
| Low volume, simple flow | Direct calls |

---

## 12. Production Checklist

```
□ Dead Letter Queue configured
□ Idempotent consumers
□ Monitoring & alerting on queue depth
□ Consumer health checks
□ Retry policy defined
□ Message expiration (TTL) set
□ Error handling and logging
□ Graceful shutdown (finish current message)
□ Poison message handling
□ Performance testing under load
```

---

## TL;DR

| Concept | Summary |
|---------|---------|
| **Queue** | Point-to-point, one consumer per message |
| **Topic** | Broadcast, all subscribers get message |
| **At-least-once** | Guaranteed delivery, handle duplicates |
| **Idempotency** | Same operation, same result (critical!) |
| **DLQ** | Where failed messages go for investigation |
| **Event** | "Something happened" — loose coupling |
| **Command** | "Do this" — direct instruction |

**Key Principles:**
- Use queues to decouple and scale
- Design for at-least-once, make operations idempotent
- Always have Dead Letter Queues
- Monitor queue depth for scaling signals
- Events for decoupling, commands for specific tasks

---

## Quick Reference

### RabbitMQ Docker

```yaml
services:
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"  # Management UI
```

### SQS Pattern

```python
import boto3

sqs = boto3.client('sqs')

# Send
sqs.send_message(QueueUrl=queue_url, MessageBody='{"task": "process"}')

# Receive
messages = sqs.receive_message(QueueUrl=queue_url, MaxNumberOfMessages=10)
for msg in messages.get('Messages', []):
    process(msg['Body'])
    sqs.delete_message(QueueUrl=queue_url, ReceiptHandle=msg['ReceiptHandle'])
```

### Interview Talking Points

1. "Queues enable async processing—user gets fast response, work happens in background"
2. "Design for at-least-once delivery and make consumers idempotent"
3. "Dead Letter Queues capture failed messages for investigation and replay"
4. "Kafka maintains order within partitions; use partition keys for related messages"
5. "Events describe what happened (loose coupling); commands tell what to do (tighter coupling)"
