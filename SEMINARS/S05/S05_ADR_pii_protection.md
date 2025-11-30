# ADR: PII Protection (Masking + Minimal Exposure)
Status: Proposed

## Context
Risk: R-04 — утечка персональных данных через ответы API  
L=4, I=4, Score=16 (Top-3)

DFD: Edge Клиент → API Gateway (API Responses, PII Exposure)  
NFR: NFR-Privacy/PII, NFR-API-Contract/Errors  

Assumptions:
- Публичные endpoints возвращают расширенные user-DTO с email/phone/address.
- В логах встречаются необработанные PII.
- Ошибки формируются вручную и иногда включают фрагменты входных данных.
- Архитектура stateless; masking можно реализовать на уровне сервисов/API Gateway.

## Decision
**Scope:**  
- Endpoints: `/api/user/*`, `/api/auth/*`, `/api/orders/*`  
- Roles: пользователи всех уровней (user/admin)  
- Layer: API Gateway + Service response DTO layer

Вводим политику защиты персональных данных:
- **Masking PII в ответах API:**  
  - email → `t***@domain.com`  
  - phone → `+7********67`  
  - address → только первые 2 компонента (город + район)
- **Response Allowlist:** ответ формируется только по `response_dto_whitelist.yaml`.
- **Error Safety:** все ошибки → RFC7807, PII запрещены в `detail`.
- **Logs:** логирование только разрешённых полей (allowlist logging), PII автоматически маскируется перед записью.
- **Retention:** хранение сырых PII в логах ≤ 30 дней (policy log storage).

## Alternatives
- Полная токенизация PII (vault) — отклонено, слишком дорогая интеграция.  
- Полное удаление всех PII из API — отклонено, ломает важные пользовательские сценарии (e-mail подтверждения, телефон).  
- Шифрование PII в ответах API — отклонено, бесполезно для клиента и увеличивает latency.

## Consequences
### Positive
+ Минимизация риска массовой утечки PII.  
+ Соответствие GDPR/152-ФЗ без изменения бизнес-логики.  
+ Единые правила формирования DTO и ошибок.

### Negative
- DTO-слой усложняется, потребуется поддержка allowlist.  
- Отладка сложнее — логи не содержат полные данные.  
- Потребуется обучение разработчиков формированию корректных error responses.

## DoD / Acceptance
Given объект пользователя:  
- email=`"testuser@mail.com"`  
- phone=`"+79991234567"`  
- address=`"Moscow, Central, Lenina 10"`  

When вызывается endpoint `/api/user/profile`  
Then:
- в ответе API:  
  - email=`"t***@mail.com"`  
  - phone=`"+7********67"`  
  - address=`"Moscow, Central"`  
- тело ошибки (при ошибке) соответствует RFC7807 и **не содержит email/phone/address**  
- в логах отсутствуют сырые значения (`testuser@mail.com`, `+79991234567`, `Lenina`)

Checks:
- test: `pii_masking_test`, `response_allowlist_test`  
- log: поиск по `"@mail.com"` → 0 результатов  
- scan/policy: PII-checker green  
- metric: `pii_exposure_incidents = 0`

## Rollback / Fallback
Отключение masking через feature-flag `pii_masking_enabled=false`, при этом аудит логов усиливается.  
Fallback: использование минимального allowlist без маскирования (Degraded Mode).

## Trace
- DFD: Клиент → API Gateway (S04_dfd.md)  
- STRIDE: Information Disclosure — “API Responses / PII Leak” (S04_stride_matrix.md)  
- Risk scoring: R-04 (Top-3) (S04_risk_scoring.md)  
- NFR: NFR-Privacy/PII, NFR-API-Contract/Errors (S03)  
- Issues: SEC-PII-001, API-DATA-CONTROL-004

## Ownership & Dates
Owner: backend-team  
Reviewers: security-team  
Date created: 2025-11-30  
Last updated: 2025-11-30

## Open Questions
- Нужен ли частичный redaction для администраторов (role-based masking)?  
- Требуется ли выделенный PII Vault в долгосрочной перспективе?
