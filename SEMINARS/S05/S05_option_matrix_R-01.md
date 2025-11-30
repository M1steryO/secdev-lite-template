# S05 — Option Matrix (R-01: SQL Injection)

## 0) Контекст риска (из S04)

**Risk-ID:** R-01
**Threat:** T (Tampering — SQL Injection)
**DFD element/edge:** Store Service → PostgreSQL (SQL Queries, PII)
**NFR link (ID):** NFR-Data-Integrity, NFR-Privacy/PII
**L×I:** L=4, I=5, Score=20

**Ограничения/предпосылки:**

- Store Service частично использует raw SQL.
- PostgreSQL содержит PII + данные заказов.
- Высокая динамика запросов (фильтры каталога/поиска).
- Требования: PCI/PII compliance, отсутствие SQL-concat в проде.
- Внедрение должно вписываться в текущий ORM-стек.

---

## 1) Критерии и шкалы (1-5)

**Польза (↑ лучше):**

- Security impact (предотвращение SQLi)
- Blast radius reduction (сужение ущерба, защита PII)

**Стоимость/сложность (↓ лучше):**

- Complexity (сложность внедрения)
- Time-to-mitigate (время до эффекта)
- Dependencies (зависимости от команд/инфры)

*Benefit = Security impact + Blast radius reduction*
*Cost = Complexity + Time-to-mitigate + Dependencies*
*Net = Benefit − Cost*

---

## 2) Таблица сравнения вариантов

| Alternative                                                     | Summary                                                                         | Security impact (↑) | Blast radius (↑) | Complexity (↓) | Time (↓) | Deps (↓) | **Benefit** | **Cost** | **Net** | Notes                                                                     |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------- | -------------------: | ----------------: | --------------: | --------: | --------: | ----------------: | -------------: | ------------: | ------------------------------------------------------------------------- |
| **A — Strict Parameterization + Canonicalization (ADR)** | Параметризация SQL, канонизация данных, SQL-lint |                    5 |                 4 |               2 |         2 |         2 |       **9** |    **6** |  **+3** | Быстро внедряется, совместимо с ORM            |
| **B — Stored Procedures Only**                           | Вся логика в хранимых процедурах                    |                    5 |                 5 |               4 |         4 |         4 |      **10** |   **12** | **−2** | Требует миграций, большой рефакторинг    |
| **C — WAF / SQL Injection Rules**                        | Фильтрация SQL-паттернов на периметре             |                    3 |                 2 |               2 |         1 |         3 |       **5** |    **6** | **−1** | Частично помогает, не защищает сервис→DB |

---

## 3) Тай-брейкеры

(между A и C, но A лучше по Net)

- **Compliance:** A покрывает требования Data Integrity / PII лучше WAF.
- **Maintainability:** A проще сопровождать (правила в коде + lint).
- **Team fit:** Команда уже использует ORM → параметризация естественна.
- **Observability:** A даёт более прозрачные логи SQL.
- **Rollback safety:** У A простой откат через feature-flag.

---

## 4) Решение (для переноса в ADR)

**Chosen alternative:** **A — Strict Parameterization + Canonicalization**

**Почему:**
Альтернатива A даёт максимальный security impact (5/5), значимо снижает blast radius и при этом требует умеренной сложности.
WAF не покрывает внутренний сервис→DB путь, а Stored Procedures слишком дорогие и долгие.
A обеспечивает быстрое снижение риска, лёгкий rollback и вписывается в текущую архитектуру ORM.

**ADR candidate:**
`SQL Injection Protection (Parameterization + Canonicalization)`

**Связки:**
Risk-ID: **R-01**
NFR-ID: **NFR-Data-Integrity, NFR-Privacy/PII**
DFD: **Store Service → PostgreSQL**

**Следующие шаги:**

1. Включить SQL-lint (запрет string concatenation).
2. Добавить canonicalization middleware (NFC/E.164/UTC).
3. Переписать legacy raw SQL в параметризованные вызовы.
4. Добавить `ApprovedSQL`-wrapper для исключений.
5. Настроить тесты `test_no_raw_sql` и `test_canonicalization`.
