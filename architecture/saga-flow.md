<div align="center">

# 🔄 Hermex Saga Flow
### Choreography-Based Distributed Transactions & Compensations

[ **English** ] &nbsp;•&nbsp; [ [Українська](saga-flow.ua.md) ] &nbsp;•&nbsp; [ [System Overview](../README.md) ]

</div>

Hermex employs the **Choreography-based Saga** pattern to handle distributed transactions across independent microservices without a centralized coordinator (Orchestrator). Services interact through asynchronous events delivered by **RabbitMQ 3.13**, ensuring loose coupling and high horizontal scalability.

---

## 🧭 1. Happy Path: Order Creation & Payment Settlement

In the standard flow, an order sequentially transitions through stock reservation and funds capture:

```mermaid
sequenceDiagram
    autonumber
    actor Client as Customer (Browser)
    participant Web as Hermex Website (:3000)
    participant Gateway as API Gateway (:4000)
    participant Order as Order Service (:50051)
    participant RMQ as RabbitMQ 3.13
    participant Inv as Inventory Service (:50053)
    participant Portal as Payment Portal (:3001)
    participant Pay as Payment Service (:50052)

    Client->>Web: Clicks "Place Order"
    Web->>Gateway: POST /api/v1/orders (with X-Idempotency-Key)
    Gateway->>Order: gRPC CreateOrder()
    Order->>Order: Save order (status: PENDING)
    Order->>RMQ: Publish "order.created" (order.topic)
    Order-->>Gateway: Order ID & Status PENDING
    Gateway-->>Web: Redirect to /pay/:orderId

    RMQ->>Inv: Consumer "order.created"
    Inv->>Inv: Reserve requested stock items
    Inv->>RMQ: Publish "inventory.reserved" (inventory.topic)

    Web->>Portal: Opens /pay/:orderId
    Client->>Portal: Enters card details & clicks "Pay"
    Portal->>Gateway: POST /api/v1/payments/process
    Gateway->>Pay: gRPC ProcessPayment()
    Pay->>Pay: Validate scenario & capture funds (Idempotent)
    Pay->>RMQ: Publish "payment.succeeded" (payment.topic)

    RMQ->>Order: Consumer "payment.succeeded"
    Order->>Order: Update status to CONFIRMED
    
    RMQ->>Gateway: Consumer "payment.succeeded"
    Gateway-->>Web: SSE Event: status = CONFIRMED
    Web-->>Client: Live Tracker timeline updates
```

---

## 🛑 2. Rollback Path 1: Insufficient Warehouse Stock (`inventory.failed`)

When items are out of stock during checkout:

```mermaid
sequenceDiagram
    autonumber
    participant Order as Order Service
    participant RMQ as RabbitMQ
    participant Inv as Inventory Service
    participant Gateway as API Gateway
    actor Client as Customer

    Order->>RMQ: Publish "order.created"
    RMQ->>Inv: Consumer "order.created"
    Inv->>Inv: Verify warehouse balance: OUT_OF_STOCK
    Inv->>RMQ: Publish "inventory.failed" (inventory.topic)
    
    RMQ->>Order: Consumer "inventory.failed"
    Order->>Order: Update status to CANCELLED
    
    RMQ->>Gateway: Consumer "inventory.failed"
    Gateway-->>Client: SSE Event: status = CANCELLED (Items out of stock)
```

---

## 💳 3. Rollback Path 2: Payment Failure & Compensating Restock

If payment is declined by the issuer bank or funds are insufficient, a compensating transaction executes to release reserved stock:

```mermaid
sequenceDiagram
    autonumber
    participant Inv as Inventory Service
    participant Pay as Payment Service
    participant RMQ as RabbitMQ
    participant Order as Order Service
    participant Gateway as API Gateway

    Note over Inv: Items previously reserved (inventory.reserved)
    Pay->>Pay: Capture failed: INSUFFICIENT_FUNDS
    Pay->>RMQ: Publish "payment.failed" (payment.topic)

    par Stock Compensation (Restock)
        RMQ->>Inv: Consumer "payment.failed"
        Inv->>Inv: Restock items back to available balance
        Inv->>RMQ: Publish "inventory.compensation.completed"
    and Order Cancellation
        RMQ->>Order: Consumer "payment.failed"
        Order->>Order: Update status: CANCELLED (reason: PAYMENT_FAILED)
    and Client Live Notification
        RMQ->>Gateway: Consumer "payment.failed"
        Gateway-->>Gateway: Broadcast SSE status CANCELLED
    end
```

---

## 📜 4. Event Contracts (`@hermex/contracts`)

All events are strictly typed and published with mandatory `trace_id` headers:

| Event / Routing Key | Exchange | Payload Interface | Purpose |
| :--- | :--- | :--- | :--- |
| `order.created` | `order.topic` | `OrderCreatedPayload` | Order Saga initiation |
| `order.cancelled` | `order.topic` | `OrderCancelledPayload` | Order cancellation notification |
| `order.expired` | `order.topic` | `OrderExpiredPayload` | 15-minute reservation TTL expiration |
| `inventory.reserved` | `inventory.topic` | `InventoryReservedPayload` | Successful inventory reservation |
| `inventory.failed` | `inventory.topic` | `InventoryFailedPayload` | Reservation failure (out of stock) |
| `inventory.compensation.completed` | `inventory.topic` | `InventoryCompensationPayload` | Warehouse restock confirmation |
| `payment.succeeded` | `payment.topic` | `PaymentSucceededPayload` | Successful charge authorization |
| `payment.failed` | `payment.topic` | `PaymentFailedPayload` | Payment decline (reason, error code) |

---

## ⚡ 5. Idempotency & Fault Tolerance

1. **Idempotency Keys:** Every order and payment request carries an `X-Idempotency-Key` (UUIDv4) header.
2. **Deduplication in Redis 7:** The payment service validates the idempotency key in Redis with a 24-hour TTL before executing transactions.
3. **Dead Letter Queue (`hermex.dlx`):** Messages that fail processing 3 times (`nack(false)`) route to `hermex.dead.letter.queue` for incident investigation.
