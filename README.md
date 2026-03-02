[![tested with jest](https://img.shields.io/badge/tested_with-jest-99424f.svg)](https://github.com/facebook/jest) [![jest](https://jestjs.io/img/jest-badge.svg)](https://github.com/facebook/jest)

```mermaid
classDiagram
    direction TB

    %% Core Services
    class OrderOrchestrator {
        -instrumentClient: IInstrumentStack
        -ledgerClient: ILedgerService
        -adapter: IProviderAdapter
        +createOrder(OrderRequest) v_order_id
        +validateTrade(Order) bool
        +processExecution(ProviderEvent)
        +handleSync()
    }

    class ExecutionHandler {
        -db: Database
        +isDuplicate(execution_id) bool
        +calculateFillDelta(Payload) delta
        +updateCostBasis(v_order_id, fill)
        +triggerLedger(v_order_id, delta)
    }

    class SSEHandler {
        -orchestrator: OrderOrchestrator
        +onMessage(json)
        +reconnect()
    }

    class ReconciliationWorker {
        -adapter: IProviderAdapter
        +pollOpenOrders()
        +syncOrderState(v_order_id)
    }

    %% Interfaces and Adapters
    class IProviderAdapter {
        <<interface>>
        +placeOrder(Order)
        +cancelOrder(externalId)
        +getOrderDetails(v_order_id)
    }

    class GTNAdapter {
        -restClient: HTTPClient
        +mapStatus(gtnCode)
        +estimateCharges(Order)
    }

    %% Data Entities
    class Order {
        +v_order_id: UUID
        +user_id: UUID
        +status: OrderStatus
        +cumulative_filled_qty: Decimal
        +avg_price: Decimal
    }

    class Transaction {
        +transaction_id: UUID
        +v_order_id: UUID
        +provider_transaction_id: String (execution_id)
        +type: Enum (TRADE, COMMISSION)
        +amount: Decimal
    }

    class UserCostBasis {
        +user_id: UUID
        +instrument_id: UUID
        +total_quantity: Decimal
        +total_cost_basis: Decimal
        +avg_buy_price: Decimal
    }

    class OrderAuditTrail {
        +v_order_id: UUID
        +from_status: Status
        +to_status: Status
        +raw_payload: JSON
    }

    %% Relationships
    OrderOrchestrator --> IProviderAdapter : delegates
    OrderOrchestrator --> ExecutionHandler : uses
    IProviderAdapter <|.. GTNAdapter : implements
    SSEHandler --> OrderOrchestrator : triggers
    ReconciliationWorker --> IProviderAdapter : polls
    ExecutionHandler --> Transaction : writes
    ExecutionHandler --> UserCostBasis : updates
    OrderOrchestrator --> OrderAuditTrail : logs
    OrderOrchestrator ..> Order : manages
```
