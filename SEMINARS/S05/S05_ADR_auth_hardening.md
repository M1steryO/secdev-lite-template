# ADR: Authentication Hardening (Rate Limit, JWT TTL, MFA)

**Status:** Proposed

---

## Context

- **Risk:** `R-02` — подбор или компрометация учётных данных (bruteforce, credential stuffing).**Likelihood = 4, Impact = 5, Score = 20 → Top-2 риск**.
- **DFD:** затрагивает:

  - Поток `Client → API Gateway` (login attempt flow),
  - Поток `Gateway → Auth Service` (проверка логина, выдача токена),
  - Поток `Client → Sensitive API` (операции, требующие повышенной защиты).
- **NFR:**

  - `NFR-003` — корректная аутентификация
  - `NFR-007` — защита от частых попыток
  - `NFR-009` — единый формат ошибок
- **Assumptions:**

  - `/api/login` открыт и без rate-limit.
  - Access tokens живут слишком долго; refresh tokens без контроля.
  - Нет MFA.
  - JWT key rotation отсутствует.
  - Stateless архитектура, нет серверных сессий.

---

## Decision

**Scope:**

- Endpoints: `/api/login`, `/api/refresh`, `/api/sensitive/*`
- Roles: все пользовательские аккаунты
- Layer: API Gateway + Auth Service

Вводится комплексное усиление аутентификации:

### 1. Access Token TTL

- **TTL:** 15–30 минут
- **Цель:** уменьшить окно атаки при компрометации токена.

### 2. Refresh Token TTL

- **TTL:** 7–14 дней
- **Политика:** строгая валидация; запрет повторного использования.

### 3. JWT Key Rotation

- **Rotation interval:** каждые 24 часа
- **Mechanism:** JWKS (`kid`) с историей ключей для валидации старых токенов.

### 4. Brute-force Rate Limit

- **Policy:** 5 попыток логина / 60 секунд на комбинацию `(IP + login)`
- **Outcome:** 6-я попытка → `429 Too Many Requests` + `Retry-After`.

### 5. MFA for Sensitive Operations

- Требуется одноразовый код (email/OTP) для:
  - смены пароля,
  - удаления аккаунта,
  - операций уровня `/api/sensitive/*`.

### 6. Unified Authentication Error Format

- Все ошибки возвращаются в формате **RFC 7807**, без stacktrace и утечек деталей.

---

## Alternatives

### A: Полный OAuth2 Authorization Server

**Отклонено:** избыточно сложно, требует отдельного контура и миграций.

### B: mTLS для клиентов

**Отклонено:** непрактично для браузеров и пользовательских устройств.

### C: Device Flow / PKCE Everywhere

**Отклонено:** не соответствует текущей модели клиентов.

---

## Consequences

### Positive

- Существенное снижение риска bruteforce и credential reuse.
- Access tokens с коротким TTL ограничивают масштаб атаки.
- Единообразные ошибки улучшают traceability и обнаружение злоупотреблений.
- MFA уменьшает вероятность компрометации учётки даже при утечке пароля.

### Negative

- Более сложный UX: пользователи чаще проходят авторизацию, требуется MFA.
- Дополнительная инфраструктура: key rotation, хранение ключей, алерты RL.
- Усложнение логики клиента (обработка refresh потока).

---

## DoD / Acceptance

**Given** 5 неуспешных login-попыток от одного `IP + login`**When** выполняется 6-я попытка в пределах 60 секунд**Then**

- возвращается **429 Too Many Requests**
- присутствует заголовок `Retry-After`
- тело ошибки соответствует RFC7807 (`type`, `title`, `status`, `detail`)
- в логах есть `correlation_id` и событие `"login_failed"`

Дополнительно:

- `exp` у access token ≤ 30 минут
- Refresh token действует 7–14 дней
- JWKS rotation фиксируется логом `"jwt_key_rotated"`
- `/api/sensitive/*` требует MFA, иначе `403` + RFC7807 body

**Checks:**

- tests: `auth_ratelimit_test`, `jwt_expiry_test`, `mfa_required_test`
- logs: события `"login_failed"`, `"mfa_challenge_sent"`, `"jwt_key_rotated"`
- scan/policy: включён rate-limit и rotation policy
- metrics: `auth.ratelimit.triggered` > 0 при тестах

---

## Rollback / Fallback

- Rate-limit и MFA управляются feature-flag’ом:`auth_hardening_enabled=false` — отключает политику.
- Token TTL можно временно вернуть к старым значениям.
- При ошибках JWKS rotation возможно откатываться к последнему рабочему ключу.

---

## Trace

- **DFD:** Client → Auth Service (S04_dfd.md)
- **STRIDE:** Spoofing / Credential brute-force (S04_stride_matrix.md)
- **Risk scoring:** R-02 (Top-2)
- **NFR:** NFR-003, NFR-007, NFR-009
- **Issues:** `AUTH-SEC-002`, `AUTH-RL-SETUP`, `JWKS-ROTATION`

---

## Ownership & Dates

- **Owner:** backend-team
- **Reviewers:** security-team
- **Date created:** 2025-11-30
- **Last updated:** 2025-11-30

---

## Open Questions

- Нужна ли server-side ревокация refresh токенов?
- Вводить ли adaptive MFA только при аномальной активности?
- Требуется ли централизованная JWKS-служба между сервисами?
