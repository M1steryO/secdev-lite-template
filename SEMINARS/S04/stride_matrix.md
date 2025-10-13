# Матрица STRIDE

| Element | Data/Boundary | Threat (S/T/R/I/D/E) | Description | NFR link (ID) | Mitigation idea (ADR later) |
|---------|---------------|--------------------|------------|---------------|-----------------------------|
| Edge: Клиент → Auth Service | Credentials / Login | S | Подбор/перехват учетных данных, подмена JWT | NFR-AuthN, NFR-RateLimit | MFA, rate-limit логинов, короткий TTL JWT + refresh |
| Edge: Клиент → Auth Service | Login Request | T | Изменение JSON запроса (логин/пароль) | NFR-InputValidation | Pydantic-схемы, HTTPS, лимиты размера тела |
| Edge: Клиент → Auth Service | Login Flow | R | Нет аудита попыток входа | NFR-Audit, NFR-Observability | Логирование с correlation_id, структурные логи |
| Edge: Клиент → Auth Service | Error Response | I | Утечка stacktrace/detail в ошибках аутентификации | NFR-API-Contract/Errors | RFC7807, унифицированные сообщения об ошибках |
| Edge: Клиент → Auth Service | Auth Endpoint | D | DoS на эндпоинты аутентификации | NFR-RateLimit, NFR-Availability | Rate-limiting по IP/логину, WAF |
| Node: Auth Service | JWT Secrets | I | Утечка секретов подписи JWT | NFR-Config/Secrets | Хранение в Vault, ротация ключей |
| Edge: Auth Service → API Gateway | JWT Token | S | Подделка/повторное использование JWT | NFR-AuthN | Проверка подписи, claims (iss, aud, exp), короткий TTL |
| Edge: Клиент → API Gateway | API Requests | S | Spoofing с украденным JWT | NFR-AuthN | Проверка JWT на каждом запросе, revocation list |
| Edge: Клиент → API Gateway | API Input | T | JSON/parameter injection в API запросах | NFR-InputValidation | Валидация схем, санитизация входных данных |
| Edge: Клиент → API Gateway | User Requests | R | Нет сквозной трассировки запросов | NFR-Observability/Logging | Correlation ID, structured logging |
| Edge: Клиент → API Gateway | API Responses | I | Утечка PII через ответы API | NFR-Privacy/PII | Маскирование PII в ответах, минимальный data exposure |
| Edge: Клиент → API Gateway | Public API | D | DoS на публичные эндпоинты | NFR-RateLimit | Rate-limiting, лимиты на размер запросов |
| Node: API Gateway | Request Routing | E | Обход проверок авторизации при маршрутизации | NFR-AuthZ/RBAC | Единая точка проверки прав доступа |
| Edge: API Gateway → Store Service | Commands/DTO | S | Отсутствие сервисной аутентификации | NFR-AuthN | mTLS или сервисные токены между компонентами |
| Edge: API Gateway → Store Service | Commands/DTO | T | Тамперинг данных при внутренней передаче | NFR-Data-Integrity | Подписи сообщений, валидация на приемнике |
| Node: Store Service | Business Logic | E | Обход бизнес-правил через API | NFR-AuthZ/RBAC | Серверные проверки прав, tenant isolation |
| Node: Store Service | Application Code | I | Утечка PII через логи приложения | NFR-Privacy/PII | Маскирование PII в логах, классификация данных |
| Edge: Store Service → PostgreSQL | SQL Queries | S | Избыточные права доступа к БД | NFR-AuthZ/RBAC | Принцип минимальных привилегий, отдельные учетки |
| Edge: Store Service → PostgreSQL | SQL Queries | T | SQL injection через динамические запросы | NFR-Data-Integrity | Параметризованные запросы, ORM с экранированием |
| Edge: Store Service → PostgreSQL | PII Data | I | PII в открытом виде в БД | NFR-Privacy/PII | Шифрование колонок, маскирование при разработке |
| Edge: Store Service → PostgreSQL | Database Queries | D | DoS через тяжелые запросы | NFR-DoS/Resilience | Индексы, лимиты на запросы, пагинация |
| Node: PostgreSQL | Database Storage | I | Несанкционированный доступ к файлам БД | NFR-Privacy/PII | Шифрование дисков, контроль доступа к бэкапам |
| Edge: Store Service → Payment Gateway | Payment Data | S | Spoofing платежного шлюза | NFR-PCI, NFR-AuthN | TLS, проверка сертификатов, pinning |
| Edge: Store Service → Payment Gateway | Payment Request | T | Изменение платежных данных в transit | NFR-PCI, NFR-Data-Integrity | HTTPS, подписи запросов |
| Edge: Store Service → Payment Gateway | Payment Operations | R | Нет аудита платежных операций | NFR-Audit, NFR-PCI | Idempotency keys, лог всех транзакций |
| Edge: Store Service → Payment Gateway | Payment Data | I | Утечка платежных данных | NFR-PCI, NFR-Privacy | Не хранить платежные данные, токенизация |
| Edge: Store Service → Payment Gateway | Payment API | D | DoS на платежные вызовы | NFR-Availability | Circuit breaker, retry с экспоненциальной задержкой |
| Edge: Store Service → Warehouse API | Inventory Data | T | Тамперинг данных инвентаря | NFR-Data-Integrity | Валидация схем ответов, checksum |
| Edge: Store Service → Warehouse API | Sync Operations | R | Нет трейса синхронизаций | NFR-Observability | Логирование синхронизаций, метрики |
| Edge: Store Service → Warehouse API | API Calls | D | Зависание вызовов к складу | NFR-Timeouts/Retry/CB | Timeouts, retry policy, circuit breaker |
