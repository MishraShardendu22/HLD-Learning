# Chapter 4: Data Flow & Communication Patterns

## Overview

This chapter explores how data flows between different parts of a distributed system — through databases, service calls (RPC and REST APIs), and asynchronous message passing. Understanding these patterns is crucial for building scalable, reliable systems.

---

## 1. Modes of Data Flow

Data travels between components in three primary ways:

### 1.1 Via Databases

- **Write**: One process writes data to the database
- **Read**: Another process reads it later
- **Asynchronous**: Reader and writer don't communicate directly
- **Persistence**: Data survives process restarts

### 1.2 Via Service Calls (Synchronous)

- **Client-server**: Client sends request, waits for response
- **Synchronous**: Client blocks until server responds
- **Examples**: REST APIs, RPC (gRPC), GraphQL

### 1.3 Via Message Passing (Asynchronous)

- **Broker**: Message queue sits between sender and receiver
- **Asynchronous**: Sender doesn't wait for receiver
- **Decoupled**: Sender and receiver don't need to know about each other
- **Examples**: RabbitMQ, Kafka, AWS SQS

---

## 2. What Are Web Services?

### Definition

**Web services** enable different systems to communicate over a network using standardized protocols (HTTP, TCP).

### Why Web Services Matter

- **System integration**: Connect websites, databases, cloud services
- **Microservices**: Break monoliths into smaller, independent services
- **API economy**: Expose functionality to third-party developers
- **Platform independence**: Language-agnostic communication

### Examples

- **Payment gateway**: Your app calls Stripe API to process payments
- **Authentication**: Login with Google/Facebook (OAuth)
- **Cloud services**: AWS, Azure, GCP APIs
- **Third-party integrations**: Twilio (SMS), SendGrid (email)

---

## 3. Protocols: Ensuring Structured Communication

### What is a Protocol?

A **protocol** defines the rules for how systems communicate — message format, request/response structure, error handling.

### Common Protocols

#### HTTP/HTTPS

- **Request-response**: Client sends request, server returns response
- **Stateless**: Each request is independent
- **Methods**: GET, POST, PUT, DELETE, PATCH
- **Use case**: REST APIs, web browsers

#### gRPC (HTTP/2 + Protobuf)

- **Binary protocol**: Uses Protobuf for encoding
- **Bidirectional streaming**: Client and server can stream data
- **Fast**: Lower latency than REST
- **Use case**: Microservices, high-performance systems

#### WebSockets

- **Full-duplex**: Bidirectional, persistent connection
- **Real-time**: Low latency for live updates
- **Use case**: Chat apps, live dashboards, gaming

#### AMQP (Advanced Message Queuing Protocol)

- **Message brokers**: RabbitMQ, Azure Service Bus
- **Reliable delivery**: Acknowledgments, retries
- **Use case**: Asynchronous task processing

---

## 4. Types of Communication

### 4.1 Synchronous Communication

#### Definition -

**Synchronous** means the sender **waits** for a response before continuing.

#### Characteristics

- **Blocking**: Caller is blocked until response arrives
- **Immediate feedback**: Know result right away
- **Tight coupling**: Sender depends on receiver being available

#### Example: REST API Call

```python
# Synchronous HTTP request
response = requests.get("https://api.example.com/users/123")
user = response.json()  # Waits here until server responds
print(user["name"])
```

#### Pros

- Simple to reason about (linear flow)
- Immediate error handling
- Easy debugging

#### Cons

- **Latency**: Slow if server takes time
- **Cascading failures**: If one service is down, callers fail
- **Resource waste**: Threads/processes blocked while waiting

---

### 4.2 Asynchronous Communication

**Asynchronous** means the sender **doesn't wait** for a response — it continues processing.

#### Characteristics -

- **Non-blocking**: Caller continues immediately
- **Deferred feedback**: Response comes later (callback, event)
- **Loose coupling**: Sender and receiver are independent

#### Example: Message Queue

```python
# Asynchronous message sending
producer.send("email-queue", {"to": "user@example.com", "subject": "Welcome"})
print("Email queued!")  # Returns immediately, email sent later
```

#### Pros -

- **Scalability**: Doesn't block on slow operations
- **Resilience**: Failures are isolated (retry later)
- **Throughput**: Handle more requests concurrently

#### Cons -

- More complex (callbacks, promises, error handling)
- Harder to debug (distributed tracing needed)
- Eventual consistency (may not have result immediately)

---

### 4.3 One-Way Communication

**Definition** -

Sender sends a message and **doesn't expect a response**.

**Examples** -

- **Fire-and-forget**: Send metrics to logging service
- **Notifications**: Push notification to mobile device
- **Events**: Publish event to message bus

#### Use Case

```python
# Fire-and-forget logging
logger.log("User 123 logged in")  # No response needed
```

---

### 4.4 Two-Way Communication

**Definition** -

Sender sends a message and **expects a response**.

**Examples** -

- **Request-response**: REST API, RPC
- **Bidirectional streaming**: WebSocket, gRPC streaming

**Use Case** -

```python
# Two-way: REST API
response = api.get_user(123)  # Expects user data back
```

---

## 5. Real-World Examples

### Example 1: OTP Verification

#### Synchronous Flow

1. User submits phone number → **API call (synchronous)**
2. Server generates OTP → **Sends SMS via Twilio API (synchronous)**
3. User enters OTP → **API verifies (synchronous)**
4. Response: "Success" or "Invalid OTP"

#### Why Synchronous?

User waits for OTP to arrive — immediate feedback needed.

---

### Example 2: Email Processing

#### Asynchronous Flow

1. User submits contact form → **Enqueues message (asynchronous)**
2. Response: "Your message has been received!" (instant)
3. Background worker picks up message → **Sends email via SendGrid**
4. Email sent (minutes later)

#### Why Asynchronous?

- Email delivery is slow (seconds to minutes)
- User doesn't need to wait
- Better user experience (instant response)

---

### Example 3: Notifications

#### Publisher-Subscriber Model

1. **Publisher**: Order service publishes "Order Placed" event
2. **Broker**: Kafka topic receives event
3. **Subscribers**:
   - Email service → Sends confirmation email
   - Inventory service → Updates stock
   - Analytics service → Records metrics

#### Why Pub-Sub?

- **Decoupling**: Order service doesn't know about subscribers
- **Scalability**: Add new subscribers without changing publisher
- **Resilience**: Subscribers can fail independently

---

## 6. Publisher-Subscriber Model (Event-Driven Architecture)

### What is Pub-Sub?

**Publishers** produce events, **subscribers** consume them. A **message broker** sits in between.

### Components

#### 1. **Publisher**

Produces events (e.g., "User Registered", "Order Placed")

#### 2. **Broker**

Message queue or event stream (Kafka, RabbitMQ, AWS SNS)

#### 3. **Subscriber**

Consumes events and takes action

### Example Architecture

```md
┌──────────────┐       ┌──────────┐       ┌────────────────┐
│ Order Service│──────▶│  Kafka   │──────▶│ Email Service  │
└──────────────┘       │ (Broker) │       └────────────────┘
                        │          │       ┌────────────────┐
                        │          │──────▶│ Inventory Svc  │
                        │          │       └────────────────┘
                        │          │       ┌────────────────┐
                        │          │──────▶│ Analytics Svc  │
                        └──────────┘       └────────────────┘
```

### Benefits

- **Loose coupling**: Services don't know about each other
- **Scalability**: Add subscribers without changing publisher
- **Reliability**: Broker ensures delivery (persistent queues)
- **Flexibility**: Easy to add new event handlers

### Challenges

- **Complexity**: Need to manage broker infrastructure
- **Debugging**: Tracing events across services
- **Ordering**: Ensuring events are processed in order
- **Exactly-once delivery**: Difficult to guarantee

---

## 7. Remote Procedure Call (RPC)

### What is RPC?

**RPC** makes a function call to a remote service look like a local function call.

### How RPC Works

#### Local Function Call

```python
# Local function
def get_user(user_id):
    return {"id": user_id, "name": "Alice"}

user = get_user(123)  # Direct call
```

#### RPC (Remote Function Call)

```python
# RPC stub (generated code)
user = user_service.get_user(123)  # Looks local, but calls remote server
```

**Behind the scenes**:

1. **Client stub** serializes arguments (Protobuf, JSON)
2. Sends request over network (HTTP/2, TCP)
3. **Server stub** deserializes arguments
4. Calls actual function on server
5. Serializes response
6. Sends response back to client
7. Client deserializes response

---

### Real-World Analogy: Ordering Pizza 🍕

#### Local Call (Cooking at Home)

You cook pizza yourself — no communication needed.

#### RPC (Ordering Pizza)

1. **You (client)**: Call pizza place (RPC call)
2. **Phone line (network)**: Transmits your order
3. **Pizza place (server)**: Receives order, makes pizza
4. **Delivery (response)**: Pizza arrives at your door

**Key point**: You don't go to the kitchen — you just call a function (`order_pizza()`), and it happens remotely.

---

### Key Benefits of RPC

#### 1. **Abstraction**

- Hide network complexity
- Call remote services like local functions
- Language-agnostic (Protobuf supports 10+ languages)

#### 2. **Simplified Communication**

- No manual HTTP requests, JSON parsing
- Generated client code (type-safe)

#### 3. **Scalability**

- Distribute work across multiple servers
- Load balance requests

---

### Challenges of RPC

#### 1. **Network Dependency**

- **Problem**: Network can fail, be slow, or timeout
- **Solution**: Retries, timeouts, circuit breakers

#### 2. **Failure Handling**

- **Problem**: Did the request fail? Did it succeed but response was lost?
- **Solution**: Idempotent operations, unique request IDs

#### 3. **Latency**

- **Problem**: Remote calls are 100-1000x slower than local calls
- **Solution**: Caching, batching, async calls

#### 4. **Versioning**

- **Problem**: Client and server may have different versions
- **Solution**: Backward/forward compatibility (Protobuf, Thrift)

---

### Practical Use Case: Food Delivery App (Zomato)

#### Services

- **User Service**: Manages user profiles
- **Restaurant Service**: Manages restaurant data
- **Order Service**: Handles order placement

#### RPC Flow

1. **User places order** (via mobile app)
2. **Order Service** (client) calls:

   ```python
   user = user_service.get_user(user_id)          # RPC call 1
   restaurant = restaurant_service.get_restaurant(restaurant_id)  # RPC call 2
   ```

3. **Services respond** with user and restaurant data
4. **Order Service** validates and creates order
5. **Response** sent to mobile app

#### Why RPC?

- **Type safety**: Protobuf ensures correct data types
- **Performance**: gRPC is faster than REST
- **Simplicity**: Call functions instead of HTTP APIs

---

## 8. Synchronous vs Asynchronous: When to Use What?

### When Synchronous is Better

#### Use Cases

- **User-facing actions**: Login, checkout, search
- **Immediate feedback needed**: Payment processing
- **Simple workflows**: Single service call

#### Example: ATM Withdrawal

1. User requests ₹1000
2. **Synchronous check**: Bank verifies balance
3. Immediate response: "Success" or "Insufficient funds"

**Why synchronous?** User can't leave without knowing result.

---

### When Asynchronous is Better

#### Use Cases -

- **Long-running tasks**: Video encoding, report generation
- **Non-critical operations**: Sending emails, logging
- **High-throughput systems**: Event processing, data pipelines

#### Example: Video Upload (YouTube)

1. User uploads video → **Immediate response**: "Upload successful!"
2. Background jobs (async):
   - Transcode video to different resolutions
   - Generate thumbnails
   - Extract metadata
   - Scan for copyright violations

**Why asynchronous?** User doesn't wait for processing (minutes to hours).

---

### Limitations of Synchronous Communication

#### Problem 1: Cascading Failures

- Service A calls Service B (sync)
- Service B calls Service C (sync)
- If C is down, B fails, then A fails
- **Solution**: Circuit breakers, fallbacks

#### Problem 2: Resource Exhaustion

- Each synchronous call holds a thread/connection
- Slow services block resources
- **Solution**: Async calls, thread pools, timeouts

#### Problem 3: Tight Coupling

- Changes to one service affect all callers
- Hard to evolve independently
- **Solution**: Asynchronous messaging, event-driven

---

## 9. Asynchronous Messaging and Message Queues

### What is a Message Queue?

A **message queue** is a buffer that stores messages between sender and receiver.

### How It Works

```md
Producer → [Message Queue] → Consumer
```

1. **Producer** sends message to queue
2. Queue stores message persistently
3. **Consumer** pulls message when ready
4. Consumer processes message
5. Consumer acknowledges (ACK) completion

---

### Benefits of Message Queues

#### 1. **Decoupling**

- Producer and consumer don't need to be online simultaneously
- Can scale independently

#### 2. **Load Leveling**

- Queue absorbs traffic spikes
- Consumers process at their own pace

#### 3. **Reliability**

- Messages persisted to disk
- Retry failed messages

#### 4. **Fault Tolerance**

- If consumer crashes, message stays in queue
- Another consumer can pick it up

---

### Example: E-commerce Order Processing

#### Without Queue (Synchronous)

```python
def place_order(order):
    create_order(order)          # DB write
    charge_payment(order)        # Payment API (slow)
    send_confirmation_email(order)  # Email service (slow)
    update_inventory(order)      # DB update
    notify_warehouse(order)      # Warehouse API
    return "Order placed!"       # User waits for ALL of this!
```

**Problem**: User waits 5-10 seconds for everything to complete.

#### With Queue (Asynchronous)

```python
def place_order(order):
    create_order(order)          # DB write
    queue.send("order-processing", order)  # Enqueue
    return "Order placed!"       # Instant response!

# Background worker
def process_order(order):
    charge_payment(order)
    send_confirmation_email(order)
    update_inventory(order)
    notify_warehouse(order)
```

**Benefit**: User gets instant feedback, processing happens in background.

---

### Popular Message Queues

| Tool | Type | Best For |
|------|------|----------|

| **RabbitMQ** | Queue | Task distribution, work queues |
| **Apache Kafka** | Event stream | High-throughput, event sourcing |
| **AWS SQS** | Queue | Simple, managed queue (AWS) |
| **Redis Streams** | Stream | Low-latency, in-memory |
| **Google Pub/Sub** | Pub/Sub | Serverless, event-driven |

---

## 10. Actor Model (Advanced)

### What is the Actor Model?

The **actor model** is a programming model where "actors" are independent units that:

- Have their own state
- Communicate via asynchronous messages
- Process one message at a time (no shared state)

**Characteristics** -

- **Concurrency**: Actors run in parallel
- **Isolation**: No shared memory (no race conditions)
- **Location transparency**: Actors can be on different machines

### Example: Akka (JVM), Erlang, Orleans (.NET)

#### Actor Example

```python
class UserActor:
    def __init__(self, user_id):
        self.user_id = user_id
        self.balance = 0
    
    def handle_message(self, msg):
        if msg["type"] == "deposit":
            self.balance += msg["amount"]
        elif msg["type"] == "withdraw":
            self.balance -= msg["amount"]
```

Each user has their own actor → no locking needed!

**Use Cases** -

- **Distributed systems**: Erlang (WhatsApp, Discord)
- **Gaming**: Real-time multiplayer games
- **IoT**: Millions of independent devices

---

## Summary

### Key Takeaways

1. **Data flows** via databases, service calls (RPC/REST), or message queues
2. **Synchronous communication** (RPC, REST) is simple but can cause cascading failures
3. **Asynchronous communication** (message queues, events) decouples services and improves scalability
4. **RPC** abstracts remote calls as local function calls (gRPC, Thrift)
5. **Pub-Sub** enables event-driven architecture (loose coupling, flexibility)
6. **Message queues** provide reliability, load leveling, and fault tolerance
7. **Actor model** is an advanced pattern for highly concurrent systems

### When to Use What

- **RPC/REST**: User-facing, request-response, immediate results
- **Message queues**: Background tasks, asynchronous workflows
- **Pub-Sub**: Event-driven, multiple consumers, loose coupling
- **Actor model**: High concurrency, stateful components

### Next Steps

- Learn about **replication and distributed systems** (Chapter 5)
- Explore **leader-based replication, failover, and consistency**

---

**Related Topics**: Microservices, Event sourcing, CQRS, Circuit breakers, gRPC
