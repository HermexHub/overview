<div align="center">

# 🔄 Розподілена Saga у Hermex
### Хореографія подій та компенсуючі транзакції

[ [English](saga-flow.md) ] &nbsp;•&nbsp; [ **Українська** ] &nbsp;•&nbsp; [ [Головний огляд](../README.ua.md) ]

</div>

Платформа Hermex використовує патерн **Choreography-based Saga** для управління розподіленими транзакціями між мікросервісами без централізованого координатора (Orchestrator). Сервіси взаємодіють через асинхронні події брокера **RabbitMQ 3.13**, що забезпечує слабку зв'язність та високу горизонтальну масштабованість.

---

## 🧭 1. Happy Path: Створення та оплата замовлення

У штатному сценарії замовлення послідовно проходить резервування на складі та списання коштів:

```mermaid
sequenceDiagram
    autonumber
    actor Client as Покупець (Браузер)
    participant Web as Hermex Website (:3000)
    participant Gateway as API Gateway (:4000)
    participant Order as Order Service (:50051)
    participant RMQ as RabbitMQ 3.13
    participant Inv as Inventory Service (:50053)
    participant Portal as Payment Portal (:3001)
    participant Pay as Payment Service (:50052)

    Client->>Web: Натискає «Оформити замовлення»
    Web->>Gateway: POST /api/v1/orders (з X-Idempotency-Key)
    Gateway->>Order: gRPC CreateOrder()
    Order->>Order: Збереження замовлення (статус: PENDING)
    Order->>RMQ: Публікація "order.created" (order.topic)
    Order-->>Gateway: Order ID та статус PENDING
    Gateway-->>Web: Перенаправлення на /pay/:orderId

    RMQ->>Inv: Consumer "order.created"
    Inv->>Inv: Резервування залишків товарів
    Inv->>RMQ: Публікація "inventory.reserved" (inventory.topic)

    Web->>Portal: Відкриття /pay/:orderId
    Client->>Portal: Введення даних картки та натискання «Сплатити»
    Portal->>Gateway: POST /api/v1/payments/process
    Gateway->>Pay: gRPC ProcessPayment()
    Pay->>Pay: Перевірка сценарію та списання (Ідемпотентно)
    Pay->>RMQ: Публікація "payment.succeeded" (payment.topic)

    RMQ->>Order: Consumer "payment.succeeded"
    Order->>Order: Оновлення статусу на CONFIRMED
    
    RMQ->>Gateway: Consumer "payment.succeeded"
    Gateway-->>Web: SSE подія: status = CONFIRMED
    Web-->>Client: Оновлення шкали трекера замовлення
```

---

## 🛑 2. Rollback Path 1: Нестача товару на складі (`inventory.failed`)

Коли на складі недостатньо залишків під час оформлення:

```mermaid
sequenceDiagram
    autonumber
    participant Order as Order Service
    participant RMQ as RabbitMQ
    participant Inv as Inventory Service
    participant Gateway as API Gateway
    actor Client as Покупець

    Order->>RMQ: Публікація "order.created"
    RMQ->>Inv: Consumer "order.created"
    Inv->>Inv: Перевірка залишків: OUT_OF_STOCK
    Inv->>RMQ: Публікація "inventory.failed" (inventory.topic)
    
    RMQ->>Order: Consumer "inventory.failed"
    Order->>Order: Оновлення статусу на CANCELLED
    
    RMQ->>Gateway: Consumer "inventory.failed"
    Gateway-->>Client: SSE подія: status = CANCELLED (Товар закінчився)
```

---

## 💳 3. Rollback Path 2: Помилка платежу та компенсація залишків

Якщо платіж відхилено банком або на рахунку недостатньо коштів, запускається компенсуюча транзакція для повернення зарезервованого товару на баланс складу:

```mermaid
sequenceDiagram
    autonumber
    participant Inv as Inventory Service
    participant Pay as Payment Service
    participant RMQ as RabbitMQ
    participant Order as Order Service
    participant Gateway as API Gateway

    Note over Inv: Товари зарезервовано (inventory.reserved)
    Pay->>Pay: Помилка авторизації: INSUFFICIENT_FUNDS
    Pay->>RMQ: Публікація "payment.failed" (payment.topic)

    par Компенсація на складі (Restock)
        RMQ->>Inv: Consumer "payment.failed"
        Inv->>Inv: Повернення товару на складський баланс
        Inv->>RMQ: Публікація "inventory.compensation.completed"
    and Скасування замовлення
        RMQ->>Order: Consumer "payment.failed"
        Order->>Order: Оновлення статусу: CANCELLED (причина: PAYMENT_FAILED)
    and Оповіщення клієнта
        RMQ->>Gateway: Consumer "payment.failed"
        Gateway-->>Gateway: Відправка SSE статусу CANCELLED
    end
```

---

## 📜 4. Контракти подій (`@hermex/contracts`)

Усі події суворо типізовані та містять обов'язковий ідентифікатор `trace_id`:

| Подія / Routing Key | Обмінник (Exchange) | Інтерфейс пейлоаду | Призначення |
| :--- | :--- | :--- | :--- |
| `order.created` | `order.topic` | `OrderCreatedPayload` | Ініціація Saga створення замовлення |
| `order.cancelled` | `order.topic` | `OrderCancelledPayload` | Сповіщення про скасування замовлення |
| `order.expired` | `order.topic` | `OrderExpiredPayload` | Закінчення 15-хвилинного TTL резерву |
| `inventory.reserved` | `inventory.topic` | `InventoryReservedPayload` | Успішне резервування товарів |
| `inventory.failed` | `inventory.topic` | `InventoryFailedPayload` | Помилка резервування (товару немає) |
| `inventory.compensation.completed` | `inventory.topic` | `InventoryCompensationPayload` | Підтвердження повернення товару на склад |
| `payment.succeeded` | `payment.topic` | `PaymentSucceededPayload` | Успішна авторизація та списання коштів |
| `payment.failed` | `payment.topic` | `PaymentFailedPayload` | Відмова платежу (код помилки) |

---

## ⚡ 5. Ідемпотентність та стійкість до відмов

1. **Ключі ідемпотентності:** Кожен запит створення замовлення та оплати супроводжується заголовком `X-Idempotency-Key` (UUIDv4).
2. **Дедуплікація в Redis 7:** Сервіс платежів перевіряє ключ у Redis із TTL 24 години перед проведенням транзакції.
3. **Dead Letter Queue (`hermex.dlx`):** Повідомлення, що завершилися збоєм 3 рази поспіль (`nack(false)`), автоматично перенаправляються в чергу `hermex.dead.letter.queue`.
