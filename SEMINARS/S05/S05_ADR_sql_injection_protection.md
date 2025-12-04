# ADR: SQL Injection Protection (Parameterization + Canonicalization)

**Status:** Proposed

---

## Context

- **Risk:** `R-01` — SQL Injection через динамические (string-concat) запросы.**Likelihood=4, Impact=5, Score=20 → Top-1.**
- **DFD:** Поток `Store Service → PostgreSQL` обрабатывает PII и данные заказов,включая `/api/orders/*`, `/api/catalog/search`, `/api/cart/*`.
- **NFR:**

  - `NFR-010` — корректность данных, отсутствие SQLi.
  - `NFR-006` — защита персональных данных в хранилище.
- **Assumptions:**

  - В некоторых местах сервис формирует SQL вручную, через конкатенацию строк.
  - PostgreSQL хранит чувствительные данные (email, phone, orders).
  - Raw SQL используется для search и order-update сценариев.

---

## Decision

**Scope:** все SQL-операции Store Service → Database.
**Layer:** repository/data-access layer + CI policies.

Внедряется строгая политика безопасной работы с SQL:

### 1. Parametrized Queries Only

- Все запросы выполняются через параметризованные вызовы ORM/driver.
- Запрещена конкатенация строк вида `"SELECT ... " + user_input`.

### 2. Static SQL Linting (CI Gate)

- В CI добавляется проверка, запрещающая:
  - использование `f"SELECT ... {var}"`,
  - concat-построение запросов,
  - динамические вставки вместо placeholder’ов (`$1`, `$2`, `?`).

### 3. Canonicalization Before Write

- входные строки → Unicode **NFC**;
- номера телефонов → **E.164**;
- время → **UTC ISO8601**;
- нормализация выполняется в service-layer перед сохранением.

### 4. Controlled Raw SQL

- Raw SQL разрешён **только** внутри `ApprovedSQL`-модулей, прошедших security-review.

---

## Alternatives

### A. Stored Procedures Only

Отклонено — повышает сложность разработки и миграций без гарантии безопасности,
если в stored procedures всё равно может быть строковая сборка.

### B. WAF SQL Rules

Отклонено — не покрывают внутренний путь *service → database*; защищают только perimeter.

---

## Consequences

### Positive

- SQL injection устраняется как класс уязвимостей.
- Повышается единообразие и предсказуемость SQL-слоя.
- Canonicalization улучшает качество данных (search, сортировка, сравнение).

### Negative

- Потребуется рефакторинг старого кода.
- CI станет строже → возможны временные блокировки MR/PR.
- Необходима дисциплина при добавлении нового raw SQL.

---

## DoD / Acceptance

**Given** входное значение содержит SQL-инъекцию:
`"' OR 1=1 --"`

**When** вызывается endpoint `/api/orders` или любой другой,
который пишет данные в БД

**Then**:

- в логах SQL присутствуют placeholders (`$1`, `$2`), а не конкатенация;
- сохранённые данные **нормализованы** (NFC, E.164, UTC);
- CI-lint не обнаруживает нарушений;
- база данных не возвращает все записи (инъекция не срабатывает).

### Checks

- `test_no_raw_sql()` — проверка placeholder'ов.
- `test_canonicalization()` — проверка нормализации.
- `sql-lint` в CI возвращает **0 нарушений**.
- Поиск по `"SELECT " +` и `f"SELECT"` → 0 результатов.
- Метрика: `security.sql_injection_incidents = 0`.

---

## Rollback / Fallback

- Временное отключение строгих правил через feature-flag`strict_sql_mode=false`, при этом нарушения продолжают логироваться.
- Fallback: полный переход на ORM-only режим.

---

## Trace

- **DFD:** Store Service → PostgreSQL (`S04_dfd.md`)
- **STRIDE:** Tampering (SQL Queries / Injection) (`S04_stride_matrix.md`)
- **Risk Scoring:** R-01, Top-1 (`S04_risk_scoring.md`)
- **NFR:** `NFR-010`, `NFR-006`
- **Issues:** `SEC-DB-001`, `QA-SQL-LINT-ENFORCE`.

---

## Ownership & Dates

- **Owner:** backend-team
- **Reviewers:** security-team
- **Date created:** 2025-11-30
- **Last updated:** 2025-11-30

---

## Open Questions

- Нужно ли включать DB-trigger’ы для автоматической нормализации?
- Требуется ли аудит всех исторических данных (PII) для приведения к NFC/E.164?
