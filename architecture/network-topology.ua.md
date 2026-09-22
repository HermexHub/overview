<div align="center">

# 🔌 Мережева топологія та матриця інфраструктури
### Порти, внутрішні протоколи та міжсервісна взаємодія

[ [English](network-topology.md) ] &nbsp;•&nbsp; [ **Українська** ] &nbsp;•&nbsp; [ [Головний огляд](../README.ua.md) ]

</div>

Цей документ описує мережеву взаємодію, протоколи, бази даних та конфігурацію брокера повідомлень мікросервісної платформи Hermex.

---

## 🧭 1. Зведена матриця портів та протоколів

| Сервіс / Компонент | Контейнер Docker | Внутрішній порт | Зовнішній хост-порт | Протокол / Призначення |
| :--- | :--- | :---: | :---: | :--- |
| **Hermex Website** | `hermex-website` | `3000` | `3000` | HTTP / Вітрина магазину |
| **Payment Portal** | `hermex-payment-portal`| `3001` | `3001` | HTTP / Hosted Checkout |
| **API Gateway** | `hermex-api-gateway` | `4000` | `4000` | HTTP / Ingress та Live SSE |
| **Order Service** | `hermex-order-service` | `50051`<br/>`3001` | *Ізольований*<br/>*Ізольований* | gRPC (Внутрішній RPC)<br/>HTTP (`/metrics`) |
| **Inventory Service**| `hermex-inventory-service`| `50053`<br/>`3002` | *Ізольований*<br/>*Ізольований* | gRPC (Внутрішній RPC)<br/>HTTP (`/metrics`) |
| **Payment Service** | `hermex-payment-service`| `50052`<br/>`3003` | *Ізольований*<br/>*Ізольований* | gRPC (Внутрішній RPC)<br/>HTTP (`/metrics`) |
| **PostgreSQL 16** | `hermex-postgres` | `5432` | `5432` | TCP / Реляційна СУБД |
| **Redis 7** | `hermex-redis` | `6379` | `6379` | TCP / Кеш та дедуплікація |
| **RabbitMQ 3.13** | `hermex-rabbitmq` | `5672`<br/>`15672` | `5672`<br/>`15672` | AMQP 0-9-1<br/>HTTP Management UI |
| **Prometheus** | `hermex-prometheus` | `9090` | `9090` | HTTP / Збір метрик |
| **Grafana** | `hermex-grafana` | `3000` | `3000` (Docker) | HTTP / Дашборди |
| **Kibana (ELK)** | `hermex-kibana` | `5601` | `5601` | HTTP / Аналіз логів |
| **Elasticsearch** | `hermex-elasticsearch` | `9200` | `9200` | HTTP / Сховище телеметрії |

> [!NOTE]
> Відповідно до принципу **Security-by-Design**, gRPC порти (`50051`, `50052`, `50053`) та внутрішні порти метрик (`3001`, `3002`, `3003`) **не прокидаються назовні на хост-машину** в Docker Compose. Доступ до них відкритий виключно всередині мостової мережі `hermex-network`.

---

## 🐘 2. Ізоляція баз даних (Database-per-Service)

Єдиний кластер PostgreSQL 16 (`docker/init.sql`) ініціалізує 4 логічно ізольовані бази даних з окремими схемами та користувачами:

```mermaid
graph TD
    subgraph PG["PostgreSQL 16 Cluster"]
        DB1[("gateway_db<br/>(Користувачі, сесії, профілі)")]
        DB2[("orders_db<br/>(Замовлення, позиції, статуси)")]
        DB3[("inventory_db<br/>(Товари, специфікації GIN, залишки)")]
        DB4[("payments_db<br/>(Транзакції, сценарії, ключі)")]
    end

    GW["API Gateway"] --> DB1
    ORD["Order Service"] --> DB2
    INV["Inventory Service"] --> DB3
    PAY["Payment Service"] --> DB4
```

---

## 🐇 3. Топологія черг та обмінників RabbitMQ

```
[Обмінники / Exchanges]
  ├── order.topic (Topic Exchange)
  │     ├── order.created
  │     ├── order.cancelled
  │     └── order.expired
  │
  ├── inventory.topic (Topic Exchange)
  │     ├── inventory.reserved
  │     ├── inventory.failed
  │     └── inventory.compensation.completed
  │
  ├── payment.topic (Topic Exchange)
  │     ├── payment.succeeded
  │     └── payment.failed
  │
  ├── notifications.fanout (Fanout Exchange)
  │     └── [Бродкаст передплатникам сповіщень]
  │
  └── hermex.dlx (Dead Letter Exchange)
        └── hermex.dead.letter.queue (Черга збійних повідомлень)
```

---

## 🌐 4. gRPC Контракти (`@hermex/contracts`)

- **`OrderGrpcService` (`order.proto` на `:50051`):**
  - `CreateOrder(CreateOrderRequest) -> CreateOrderResponse`
  - `GetOrderById(GetOrderByIdRequest) -> OrderMessage`
  - `CancelOrder(CancelOrderRequest) -> CancelOrderResponse`
- **`InventoryGrpcService` (`inventory.proto` на `:50053`):**
  - `GetProducts(GetProductsRequest) -> GetProductsResponse`
  - `GetProductById(GetProductByIdRequest) -> GetProductByIdResponse`
  - `ValidateCart(ValidateCartRequest) -> ValidateCartResponse`
- **`PaymentGrpcService` (`payment.proto` на `:50052`):**
  - `CreatePaymentSession(CreatePaymentSessionRequest) -> CreatePaymentSessionResponse`
  - `ProcessPayment(ProcessPaymentRequest) -> ProcessPaymentResponse`
