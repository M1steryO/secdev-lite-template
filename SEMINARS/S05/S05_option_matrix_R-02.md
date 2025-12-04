# S05 — Option Matrix (R-02: Auth Hardening)

## 0) Контекст риска (из S04)

**Risk-ID:** R-02
**Threat:** S (Spoofing — подбор/кража учетных данных)
**DFD element/edge:** Клиент → Auth Service (Login Flow)
**NFR link (ID):** NFR-003, NFR-007, NFR-009
**L×I:** L=4, I=5, Score=20

**Ограничения/предпосылки:**

- Публичный endpoint `/api/login` без ограничений частоты.
- JWT с длинным TTL и отсутствием key rotation.
- Нет MFA.
- Архитектура stateless (нет server-side sessions).
- Требования безопасности: защита от bruteforce, credential stuffing, replay.

---

## 1) Критерии и шкалы (1-5)

**Польза (↑ лучше):**

- Security impact (насколько снижает риск)
- Blast radius reduction (уменьшение последствий компрометации)

**Стоимость/сложность (↓ лучше):**

- Complexity
- Time-to-mitigate
- Dependencies

*Benefit = Security impact + Blast radius reduction*
*Cost = Complexity + Time-to-mitigate + Dependencies*
*Net = Benefit − Cost*

---

## 2) Таблица сравнения вариантов

| Alternative                                                        | Summary                                                                 | Security impact (↑) | Blast radius (↑) | Complexity (↓) | Time (↓) | Deps (↓) | **Benefit** | **Cost** | **Net** | Notes                                                                            |
| ------------------------------------------------------------------ | ----------------------------------------------------------------------- | -------------------: | ----------------: | --------------: | --------: | --------: | ----------------: | -------------: | ------------: | -------------------------------------------------------------------------------- |
| **A — JWT TTL + Refresh + Key Rotation + Rate-Limit (ADR)** | Короткий TTL, refresh flow, rotation, RL                        |                    5 |                 4 |               2 |         2 |         2 |       **9** |    **6** |  **+3** | Быстро внедряется, минимальные зависимости |
| **B — Full OAuth2 Authorization Server**                    | Внешний IdP, OAuth2/OIDC, device flow                            |                    5 |                 5 |               5 |         4 |         5 |      **10** |   **14** | **−4** | Огромная сложность, долгий rollout                        |
| **C — mTLS client authentication**                          | Аутентификация клиента по сертификату |                    4 |                 4 |               4 |         4 |         5 |       **8** |   **13** | **−5** | Не работает для браузеров, high ops load                   |

---

## 3) Тай-брейкеры при равенстве Net

(нет равенства, но фиксируем аргументацию)

- **Compliance:** A удовлетворяет требованиям аутентификации без изменения UX.
- **Maintainability:** A проще всего поддерживать, JWT rotation — стандартная практика.
- **Team fit:** команда уже использует JWT; OAuth2 и PKI потребуют больших усилий.
- **Rollback safety:** A легко отключить через feature-flags.
- **Observability:** A даёт предсказуемые login-failure и RL метрики.

---

## 4) Решение (для переноса в ADR)

**Chosen alternative:** **A — JWT TTL + Refresh + Key Rotation + Rate-Limit**

**Почему:**
Альтернатива A обеспечивает максимальное снижение риска bruteforce и credential compromise при средней сложности внедрения.
OAuth2/PKI дают чуть больший security impact, но их стоимость, время внедрения и количество зависимостей несопоставимо выше.
A быстро приносит эффект, совместима со stateless архитектурой и безопасно откатывается.

**ADR candidate:**
`Auth Hardening (MFA + JWT TTL + Brute-Force Protection)`

**Связки:**
Risk-ID: **R-02**
NFR-ID: **NFR-003, NFR-007, NFR-009**
DFD: **Client → Auth Service**

**Следующие шаги:**

1. Настроить RL: 5 попыток / 60 сек per IP+login.
2. Ввести access TTL 15–30m + refresh TTL 7–14d.
3. Добавить JWKS rotation (`kid`) каждые 24h.
4. Реализовать MFA для `/api/sensitive/*`.
5. Унифицировать ошибки аутентификации в формате RFC7807.
