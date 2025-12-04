# S05 — Option Matrix (R-04: PII Protection)

## 0) Контекст риска (из S04)

**Risk-ID:** R-04
**Threat:** I (Information Disclosure — утечка PII через ответы API)
**DFD element/edge:** Клиент → API Gateway (API Responses / PII Exposure)
**NFR link (ID):** NFR-006, NFR-009
**L×I:** L=4, I=4, Score=16

**Ограничения/предпосылки:**

- User/profile/order endpoints возвращают избыточную PII.
- В логах встречается необработанный email/phone/address.
- Ошибки формируются вручную и могут включать PII.
- Требования: GDPR/152-ФЗ, минимизация data exposure.
- Система stateless, PII не должно попадать в логи/ответы без маскирования.

---

## 1) Критерии и шкалы (1-5)

**Польза (↑ лучше):**

- Security impact (снижение вероятности/масштаба утечки)
- Blast radius reduction (локализация ущерба)

**Стоимость/сложность (↓ лучше):**

- Complexity (внедрение/поддержка)
- Time-to-mitigate (скорость эффекта)
- Dependencies (другие команды/хранилища)

*Benefit = Security impact + Blast radius reduction*
*Cost = Complexity + Time-to-mitigate + Dependencies*
*Net = Benefit − Cost*

---

## 2) Таблица сравнения вариантов

| Alternative                                                         | Summary                                                      | Security impact (↑) | Blast radius (↑) | Complexity (↓) | Time (↓) | Deps (↓) | **Benefit** | **Cost** | **Net** | Notes                                                              |
| ------------------------------------------------------------------- | ------------------------------------------------------------ | -------------------: | ----------------: | --------------: | --------: | --------: | ----------------: | -------------: | ------------: | ------------------------------------------------------------------ |
| **A — Masking + Response Allowlist + PII-Free Errors (ADR)** | Маскирование PII, allowlist полей, RFC7807  |                    4 |                 4 |               2 |         2 |         2 |       **8** |    **6** |  **+2** | Быстро внедряется, соответствует GDPR |
| **B — PII Tokenization / Vault**                             | Хранение PII в secure vault, токенизация |                    5 |                 5 |               5 |         4 |         5 |      **10** |   **14** | **−4** | Очень безопасно, но долго и дорого     |
| **C — Полный отказ от PII в API**              | API не возвращает e-mail/phone/address           |                    5 |                 5 |               3 |         3 |         3 |      **10** |    **9** |  **+1** | Ломает UX, требует переделки фронта    |

---

## 3) Тай-брейкеры при равенстве Net

(у A и C нет равенства, но аргументы фиксируем)

- **Compliance:** A покрывает требования GDPR без нарушения UX.
- **Maintainability:** allowlist + masking проще, чем PII vault или пересмотр API контракта.
- **Team fit:** команда уже использует DTO-слой → masking внедряется естественно.
- **Rollback safety:** A можно отключить флагом; vault или redesign — нет.
- **Observability:** A позволяет логировать безопасные данные (masked), сохраняя качество диагностики.

---

## 4) Решение (для переноса в ADR)

**Chosen alternative:** **A — Masking + Response Allowlist + PII-Free Errors**

**Почему:**
A даёт высокий уровень защиты (4/5) при минимальной сложности внедрения.
Tokenization (B) слишком дорог и требует отдельных хранилищ/команд.
Полный отказ от PII (C) нарушает UX и требует переработки многих клиентских сценариев.
A обеспечивает быструю, проверяемую и обратимую меру защиты, совместимую с GDPR/152-ФЗ.

**ADR candidate:**
`PII Protection (Masking + Minimal Exposure)`

**Связки:**
Risk-ID: **R-04**
NFR-ID: **NFR-006, NFR-009**
DFD: **Client → API Gateway**

**Следующие шаги:**

1. Реализовать masking для email/phone/address.
2. Ввести `response_dto_whitelist.yaml` для публичных API.
3. Включить PII-free RFC7807 для ошибок.
4. Добавить allowlist logging (маскирование в логах).
5. Настроить retention ≤ 30 дней для PII в лог-хранилище.
