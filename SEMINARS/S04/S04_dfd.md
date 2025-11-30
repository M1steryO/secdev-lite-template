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
  %% Authentication
  C -->|"Login / Credentials / HTTPS <br/>[NFR-AuthN, NFR-TLS]"| AUTH
  AUTH -->|"JWT Token <br/>[NFR-AuthN, TokenLifetime]"| C

  %% Core API calls
  C -->|"API Requests / JWT <br/>[NFR-RateLimit, NFR-Observability]"| API
  API -->|"Commands / DTO <br/>[NFR-InputValidation, NFR-Logging]"| SVC

  %% Database
  SVC -->|"SQL Queries / Orders / PII <br/>[NFR-Data-Integrity, NFR-Privacy]"| DB

  %% External integrations
  SVC -->|"Payment Request / Callback <br/>[NFR-PCI, NFR-Idempotency, NFR-Retry]"| PAY
  SVC -->|"Inventory Sync (HTTP) <br/>[NFR-Reliability, NFR-Retry]"| WH

  %% --- Границы доверия ---
  classDef boundary fill:#f6f6f6,stroke:#999,stroke-width:1px;
  class Internet,FrontendAPI,CoreService,External boundary;
