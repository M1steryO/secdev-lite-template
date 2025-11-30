# S03 - NFR (Given-When-Then)

Этот файл - рабочий **реестр NFR** на семинаре.
Заполняйте его у себя в репозитории в: `SEMINARS/S03/S03_register.md`.

---

## Поля реестра (data dictionary)

* **ID** - короткий идентификатор, например `NFR-001`.
* **User Story / Feature** - к какой истории/фиче относится требование (ссылка допустима).
* **Category** - выберите из банка (напр.: `Performance`, `Security-AuthZ/RBAC`, `RateLimiting`, `Privacy/PII`, `Observability/Logging`, …).
* **Requirement (NFR)** - *измеримое* требование (числа/пороги/границы действия).
* **Rationale / Risk** - зачем это нужно, какой риск/ценность покрываем.
* **Acceptance (G-W-T)** - проверяемая формулировка: *Given … When … Then …*.
* **Evidence (test/log/scan/policy)** - чем подтвердим выполнение (тест, лог-шаблон, сканер, политика).
* **Trace (issue/link)** - ссылка на задачу, обсуждение, артефакт.
* **Owner** - ответственный.
* **Status** - `Draft` | `Proposed` | `Approved` | `Implemented` | `Verified`.
* **Priority** - `P1 - High` | `P2 - Medium` | `P3 - Low`.
* **Severity** - `S1 - Critical` | `S2 - Major` | `S3 - Minor`.
* **Tags** - произвольные метки (через запятую).

---

## Таблица реестра NFR (US-001 – Регистрация пользователя)

| ID                | User Story / Feature                                      | Category                               | Requirement (NFR)                                                                                                                                             | Rationale / Risk                                                         | Acceptance (G-W-T)                                                                                                                                                                                                                                                                | Evidence (test/log/scan/policy)                                        | Trace (issue/link) | Owner        | Status | Priority     | Severity    | Tags                |
| ----------------- | --------------------------------------------------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- | ------------------ | ------------ | ------ | ------------ | ----------- | ------------------- |
| **NFR-001** | US-001 – Регистрация пользователя | **Performance**                  | Время отклика `/api/register` ≤ 400 мс при 20 RPS в течение 5 мин                                                              | UX/SLO — комфортная скорость регистрации   | **Given** сервис здоров <br />**When** 20 RPS на `/api/register` 5 мин <br />**Then** P95 ≤ 400 мс, ошибки ≤ 1 %                                                                                                                   | test:`load-register-20rps`; metrics:`http_server_requests_seconds` |                    | backend-team | Draft  | P2 - Medium  | S2 – Major | perf,slo            |
| **NFR-002** | US-001 – Регистрация пользователя | **Security-InputValidation**     | Размер тела POST ≤ 1 MiB; только JSON; запрещены лишние поля                                                              | Защита от DoS и грязных данных                     | **Given** тело 2× больше 1 MiB и лишние поля <br />**When** POST `/api/register` <br />**Then** 413 или 400 в формате RFC 7807                                                                                               | e2e:`register-input-limit`; schema validator                         |                    | backend-team | Draft  | P2 – Medium | S2 – Major | validation,security |
| **NFR-003** | US-001 – Регистрация пользователя | **Security-AuthZ/RBAC**          | Незарегистрированный пользователь не может выполнять действия, требующие роли user/admin | Минимизация привилегий                              | **Given** отсутствует JWT <br />**When** POST `/api/profile` <br />**Then** 401 в формате RFC 7807 без утечки данных                                                                                                        | e2e:`unauth-access-deny`; auth-policy.md                             |                    | backend-team | Draft  | P2 - Medium  | S2 - Major  | security,auth       |
| **NFR-004** | US-001 – Регистрация пользователя | **Security-Secrets**             | В репозитории 0 секретов; используются только переменные окружения или KMS                        | Предотвращение утечек                                | **Given** репозиторий и его история <br />**When** выполняется скан секретов <br />**Then** findings = 0; конфиг читает секреты из env/KMS                                                    | report:`trufflehog-scan`; runtime config                             |                    | devops       | Draft  | P2 - Medium  | S2 - Major  | secrets,security    |
| **NFR-005** | US-001 – Регистрация пользователя | **Maintainability/FeatureFlags** | Возможность включать/выключать капчу через feature-flag без релиза                                             | Безопасные релизы и откаты                        | **Given** флаг `captcha_enabled` <br />**When** он переключается в runtime <br />**Then** поведение меняется без рестарта; логируется actor и время                                           | config:`feature-flags.yaml`; лог переключения         |                    | backend-team | Draft  | P2 - Medium  | S2 - Major  | feature,ops         |
| **NFR-006** | US-001 – Регистрация пользователя | **Privacy/PII**                  | Логи не содержат email/телефон; PII удаляются через 30 дней                                                            | Комплаенс и конфиденциальность               | **Given** DTO с PII <br />**When** логируется регистрация <br />**Then** PII маскированы; retention ≤ 30 дней                                                                                                             | log template; retention policy                                         |                    | backend-team | Draft  | P2 – Medium | S2 – Major | privacy,pii         |
| **NFR-007** | US-001 – Регистрация пользователя | **RateLimiting**                 | Лимит: 10 запросов в мин с IP; превышение → 429 + Retry-After                                                                    | Защита от спама и brute-force                              | **Given** активный IP <br />**When** > 10 POST `/api/register` за 60 с <br />**Then** 429 и корректный заголовок Retry-After                                                                                                   | e2e:`register-rate-limit`; logs 429                                  |                    | backend-team | Draft  | P2 – Medium | S2 – Major | security,rate       |
| **NFR-008** | US-001 – Регистрация пользователя | **Availability/Health**          | Через 10 с после старта `/health/ready` отвечает 200                                                                               | Готовность для проб деплоя                        | **Given** контейнер запущен <br />**When** проходит 10 с <br />**Then** GET `/health/ready` → 200 и зависимости healthy                                                                                                 | readiness/startup probe config                                         |                    | devops       | Draft  | P2 – Medium | S2 - Major  | health,ops          |
| **NFR-009** | US-001 – Регистрация пользователя | Observability/Logging    | Все HTTP-запросы логируются в JSON; обязательные поля: timestamp, method, path, status, duration_ms, correlation_id       | Трассировка и диагностика         | **Given:** запрос с/без X-Correlation-ID <br> **When:** проходит через сервис <br> **Then:** в логах валидный JSON с correlation_id и ключевыми полями            | log-format.json; e2e:logging-correlation-id; query screenshot |     | backend-team | Draft  | P2 – Medium | S2 – Major | logs,observability   |
| **NFR-010** | US-001 – Регистрация пользователя | Data-Integrity           | Все SQL запросы параметризованы; данные нормализованы: строки → NFC, телефон → E.164, время → UTC                         | Целостность и защита от инъекций  | **Given:** нестандартные данные <br> **When:** запись в БД <br> **Then:** в БД: телефон в E.164, строки в NFC, timestamps в UTC; SQL без конкатенации             | unit:normalize-*; sql-lint:no-raw-sql; ORM sample  |     | backend-team | Draft  | P2 – Medium | S2 – Major | data,integrity       |

---

## Памятка по заполнению

* **Измеримость.** В `Requirement` фиксируйте числа и границы (мс, RPS, минуты, MiB, коды 4xx/5xx, CVSS).
* **Проверяемость.** В `Acceptance (G-W-T)` используйте объективные условия и наблюдаемые факты (код ответа, квантиль, наличие заголовка, запись в лог).
* **Связность.** Сверяйте, чтобы NFR не конфликтовали (timeouts vs retry, rate limits vs SLO, privacy vs logging).
* **План проверки.** В `Evidence` укажите, чем это будет подтверждаться позже (test/log/scan/policy). В рамках семинара **реальные артефакты не требуются**.
* **Трассировка.** В `Trace` добавляйте ссылки на Issues/документы, чтобы потом не искать контекст.

---

## После семинара

* Перенесите/доработайте **8-10 утверждённых NFR** (по сути, те же строки) в раздел **NFR** вашего `GRADING/TM.md`.
* На S04-S05 свяжете эти NFR с угрозами (STRIDE) и ADR - по ID.
