# ADR: Auth Hardening (MFA + JWT TTL + Brute-Force Protection)
Status: Proposed

## Context
Risk: R-02 — подбор/кража учетных данных (bruteforce / credential stuffing)  
L=4, I=5, Score=20 (Top-2)

DFD: Edge Клиент → Auth Service (Login flow, Credentials)  
NFR: NFR-AuthN, NFR-RateLimit, NFR-API-Contract/Errors  

Assumptions:
- Endpoint `/api/login` публичен и без ограничений по частоте.
- Используются JWT access tokens без короткого TTL и без key rotation.
- MFA отсутствует, ошибки возвращаются в разных форматах.
- Stateless архитектура: Auth Service и gateway не хранят сессии.

## Decision
**Scope:**  
- Endpoints: `/api/login`, `/api/refresh`, `/api/sensitive/*`  
- Roles: все user-аккаунты  
- Layer: API Gateway + Auth Service  

Вводим политику усиленной аутентификации:
- **Access Token TTL:** 15–30 минут.  
- **Refresh Token TTL:** 7–14 дней.  
- **Key Rotation:** автоматическая ротация JWT signing keys (JWKS, `kid`) каждые 24 часа.  
- **Brute-force Rate Limit:** 5 login attempts / 60 сек (per IP + login).  
- **MFA:** email/OTP-подтверждение для `/api/sensitive/*` (смена пароля, удаление аккаунта).  
- **Unified Errors:** все ошибки аутентификации возвращаются в RFC7807 формате, без stacktrace.

## Alternatives
- Полный OAuth2 Authorization Server — отклонено (слишком тяжёлая интеграция).  
- MTLS для клиентских приложений — отклонено (не поддерживается браузерами).  
- Device Flow — отклонено (нет необходимости для текущей модели клиентов).

## Consequences
### Positive
+ Сильно снижает риск bruteforce и credential reuse.  
+ Reduce attack window: короткий срок жизни access token.  
+ Централизованный, предсказуемый формат ошибок.

### Negative
- Усложнение UX: MFA и повторный логин при expiry.  
- Необходимость поддерживать JWKS rotation и storage.  
- Требуются алерты на превышение лимитов логинов.

## DoD / Acceptance
Given 6 login-запросов с неверным паролем от одного IP+логина  
When они выполняются в течение 60 секунд  
Then:
- 6-й запрос возвращает **429 Too Many Requests**  
- в ответе есть заголовок `Retry-After`  
- тело соответствует RFC7807 (`type`, `title`, `status`, `detail`)  
- в логах присутствует `correlation_id` и запись `"login_failed"`  

Дополнительные проверки:
- access token exp ≤ 30 минут  
- refresh token exp в пределах 7–14 дней  
- JWKS rotation отображается в логах  
- MFA required для `/api/sensitive/*`

Checks:
- test: `auth_ratelimit_test`, `jwt_expiry_test`, `mfa_required_test`  
- log: `"login_failed"` + correlation_id, `"jwt_key_rotated"`  
- scan/policy: JWT key rotation enabled, RL-policy active  
- metric: `auth.ratelimit.triggered > 0` (интеграционный тест)

## Rollback / Fallback
Отключение rate-limit и MFA через feature-flag `auth_hardening_enabled=false`.  
Fallback: возвращение к старой политике TTL, включая мониторинг growth login-failure rate.

## Trace
- DFD: Клиент → Auth Service (S04_dfd.md)  
- STRIDE: Spoofing — Row "Credentials Brute-force" (S04_stride_matrix.md)  
- Risk scoring: R-02 (Top-2) (S04_risk_scoring.md)  
- NFR: NFR-AuthN, NFR-RateLimit, NFR-API-Contract (S03)  
- Issues: AUTH-SEC-002, OPS-JWKS-ROTATION, AUTH-RL-SETUP

## Ownership & Dates
Owner: backend-team  
Reviewers: security-team  
Date created: 2025-11-30  
Last updated: 2025-11-30

## Open Questions
- Требуется ли хранить refresh tokens server-side для полного revocation?  
- Нужен ли adaptive MFA (только при аномальной активности)?
