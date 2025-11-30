# S04 - DFD

```mermaid
flowchart LR
  %% --- DFD ---
  subgraph Internet[Интернет / Клиенты]
    C[Клиент: Браузер / Моб. приложение]
  end

  subgraph FrontendAPI[Frontend/API Layer]
    API[API Gateway / Controller]
    AUTH[Auth Service]
  end

  subgraph CoreService[Сервис - Магазин]
    SVC[Store Service: Catalog / Cart / Orders]
    DB[(PostgreSQL / PII)]
  end

  subgraph External[Внешние сервисы]
    PAY[Payment Gateway]
    WH[Warehouse API]
  end

  %% --- Потоки данных ---
  C -- "Login / Credentials / HTTPS [NFR-AuthN, TLS]" --> AUTH
  AUTH --> |"JWT Token [NFR-AuthN, TokenLifetime]"| C
  C -- "API Requests / JWT [NFR-RateLimit, Observability]" --> API
  API -->|"Commands / DTO [NFR-InputValidation, Logging]"| SVC
  SVC -->|"SQL / Orders / PII [NFR-Data-Integrity, Privacy/PII]"| DB
  SVC -->|"HTTP Payment / Callback [NFR-PCI, Idempotency, Retry]"| PAY
  SVC -->|"HTTP Inventory Sync [NFR-Reliability, Retry]"| WH

  %% --- Границы доверия ---
  classDef boundary fill:#f6f6f6,stroke:#999,stroke-width:1px;
  class Internet,FrontendAPI,CoreService,External boundary;
