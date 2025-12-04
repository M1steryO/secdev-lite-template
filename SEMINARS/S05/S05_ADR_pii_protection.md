# ADR: PII Protection (Masking + Controlled Exposure)

**Status:** Proposed

---

## Context

- **Risk:** `R-04` — раскрытие персональных данных через API-ответы.**Likelihood=4, Impact=4, Score=16 → входит в Top-3 рисков.**
- **DFD:** Затрагивает поток `Client → API Gateway → Services`, включая:

  - Response DTO,
  - Error responses,
  - Логи уровня Gateway/Service.
- **NFR:**

  - `NFR-006` — защита персональных данных,
  - `NFR-009` — безопасный и предсказуемый формат ошибок.
- **Assumptions:**

  - Публичные endpoints возвращают расширенные DTO с email/phone/address.
  - В логах встречаются сырые PII.
  - Ошибки формируются вручную и не фильтруют входные данные.
  - Архитектура stateless → masking можно устанавливать на Gateway/DTO-layer.

---

## Decision

**Scope:**

- **Endpoints:** `/api/user/*`, `/api/auth/*`, `/api/orders/*`
- **Roles:** user/admin
- **Layer:** API Gateway + DTO serialization layer

Вводится единая политика обработки PII на уровне API:

### 1. Masked PII в ответах API

- **email:** `testuser@mail.com` → `t***@mail.com`
- **phone:** `+79991234567` → `+7********67`
- **address:** только первые два компонента (`город + район`), без улицы и номера.

### 2. Response Allowlist

Все возвращаемые поля разрешаются только через
`response_dto_whitelist.yaml`.
Никакие PII не попадут в ответ, если они не перечислены явно.

### 3. Error Safety (RFC7807)

- Ошибки возвращаются в стандартизированном формате.
- **PII в `detail` и `instance` запрещены.**
- Ошибки, сформированные вручную, проходят через фильтр redaction.

### 4. Log Redaction / Allowlist Logging

- До записи в лог PII автоматически маскируются.
- Разрешены только поля из лог-allowlist.
- Политика хранения сырых логов (если встречаются) ≤ 30 дней.

---

## Alternatives

### A. Vault Tokenization

Отклонено — высокая стоимость внедрения, сильно усложняет интеграцию.

### B. Полное исключение PII из API

Отклонено — ломает UX (email подтверждения, телефонное восстановление).

### C. Шифрование PII в ответах API

Отклонено — бесполезно для клиента и увеличивает latency.

---

## Consequences

### Positive

- Значительное снижение риска массовой утечки PII.
- Соответствие требованиям GDPR/152-ФЗ.
- Централизованная политика формирования DTO и ошибок.

### Negative

- DTO-слой усложняется за счёт allowlist.
- Отладка осложняется минимизацией логов.
- Требуется обучение команды по формированию безопасных error responses.

---

## DoD / Acceptance

**Given** объект пользователя:

- `email = "testuser@mail.com"`
- `phone = "+79991234567"`
- `address = "Moscow, Central, Lenina 10"`

**When** вызывается `/api/user/profile`

**Then**:

- в ответе API:
  - `email = "t***@mail.com"`
  - `phone = "+7********67"`
  - `address = "Moscow, Central"`
- RFC7807-ошибка **не содержит** email/phone/address
- в логах отсутствуют сырые строки
  (`testuser@mail.com`, `+79991234567`, `"Lenina"`)

### Checks

- **tests:** `pii_masking_test`, `response_allowlist_test`
- **logs:** поиск по `"@mail.com"` → 0 результатов
- **scan/policy:** PII-checker green
- **metric:** `pii_exposure_incidents = 0`

---

## Rollback / Fallback

- Masking отключается feature-flag’ом:`pii_masking_enabled = false`.
- В режиме degraded-mode используется минимальный allowlist без masking.
- При rollback усиливается аудит логов.

---

## Trace

- **DFD:** Edge → API Gateway → Services (`S04_dfd.md`)
- **STRIDE:** Information Disclosure — `API Response / PII` (`S04_stride_matrix.md`)
- **Risk scoring:** `R-04`, Top-3 (см. `S04_risk_scoring.md`)
- **NFR:** NFR-006, NFR-009
- **Issues:** `SEC-PII-001`, `API-DATA-CONTROL-004`

---

## Ownership & Dates

- **Owner:** backend-team
- **Reviewers:** security-team
- **Date created:** 2025-11-30
- **Last updated:** 2025-11-30

---

## Open Questions

- Нужен ли role-based masking (админы видят больше)?
- Требуется ли в будущем выделенный PII-Vault?
