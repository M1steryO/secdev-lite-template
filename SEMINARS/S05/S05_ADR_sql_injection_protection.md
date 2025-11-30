# ADR: SQL Injection Protection (Parameterization + Canonicalization)
Status: Proposed

## Context
Risk: R-01 — SQL injection через динамические запросы  
L=4, I=5, Score=20 (Top-1)

DFD: Edge Store Service → PostgreSQL (SQL / PII)  
NFR: NFR-Data-Integrity, NFR-Privacy/PII  

Assumptions:
- Store Service формирует часть SQL-запросов вручную.
- PostgreSQL хранит PII и данные заказов.
- Операции: создание/обновление заказов, фильтрация каталога, просмотр корзины.

## Decision
**Scope:** Все запросы Store Service → DB:  
- `/api/orders/*` (create/update/view)  
- `/api/catalog/search`  
- `/api/cart/*`  
**Layer:** service layer (репозитории), DB policy

Вводим политику защиты:
- **Param Query Only:** все SQL — только через параметризованные вызовы ORM/driver.  
- **Static SQL lint:** CI блокирует string concatenation в SQL.  
- **Canonicalization:**  
  - строки → Unicode NFC  
  - телефон → E.164  
  - время → UTC  
- **Raw SQL** допускается только через `ApprovedSQL` с ревью.

## Alternatives
- Stored Procedures Only — отклонено (слишком тяжёлые миграции).  
- WAF SQL-rules — отклонено (не защищает внутренний сервис → DB путь).

## Consequences
### Positive
+ Устраняет риск SQL injection.  
+ Единая канонизация данных.

### Negative
- Требует обновления старого кода и настройки CI.  
- Вводит дополнительную дисциплину при работе с DB.

## DoD / Acceptance
Given входные данные содержат payload вида `"' OR 1=1 --"`  
When Store Service выполняет запись в БД по `/api/orders`  
Then:
- в логах запроса отображается SQL с placeholder (`$1`, `$2`) вместо string concat;  
- сохранённые данные нормализованы (NFC, E.164, UTC).

Checks:
- test: `test_no_raw_sql()`, `test_canonicalization()`  
- scan/policy: `sql-lint` показывает 0 нарушений  
- log: поиск по `"SELECT " +` → 0 результатов  
- metric: `security.sql_injection_incidents = 0`

## Rollback / Fallback
Отключение strict режима через флаг `strict_sql_mode=false` с сохранением логирования нарушений.

## Trace
- DFD: Store Service → PostgreSQL (S04_dfd.md)  
- STRIDE: Tampering — Row "SQL Queries / Injection" (S04_stride_matrix.md)  
- Risk scoring: R-01 (Top-1) (S04_risk_scoring.md)  
- NFR: NFR-Data-Integrity, NFR-Privacy/PII (S03)  
- Issues: SEC-DB-001, QA-SQL-LINT-ENFORCE

## Ownership & Dates
Owner: backend-team  
Reviewers: security-team  
Date created: 2025-11-30  
Last updated: 2025-11-30

## Open Questions
- Нужны ли DB-triggers для доп. нормализации?
