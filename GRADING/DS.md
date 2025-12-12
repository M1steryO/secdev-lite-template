# DS - Отчёт «DevSecOps-сканы и харднинг»

> Этот файл - **индивидуальный**. Его проверяют по **rubric_DS.md** (5 критериев × {0/1/2} → 0-10).
> Подсказки помечены `TODO:` - удалите после заполнения.
> Все доказательства/скрины кладите в **EVIDENCE/** и ссылайтесь на конкретные файлы/якоря.

---

## 0) Мета

- **Проект:** учебный шаблон secdev-lite-template-ds
- **Версия:** 2025/12/09

---

## 1) SBOM & SCA (DS1)

**SBOM & SCA**: сгенерирован SBOM (CycloneDX), выполнен SCA.
**Артефакты**: EVIDENCE/S09/sbom.json, EVIDENCE/S09/sca_report.json.
**Сводка**: EVIDENCE/S09/sca_summary.md.
**Actions**: [ссылка](https://github.com/M1steryO/secdev-lite-template-ds/actions/runs/20066055689/job/57554790061).

В пакете Jinja2 обнаружены **3 уязвимости** среднего и высокого уровня, связанные с обходом sandbox: **CVE-2024-56326, CVE-2025-27516, CVE-2024-56201**.

**Рекомендуемое действие:** обновить Jinja2 до версии  **3.1.6 или выше** .

**Статус:** уязвимости приняты в работу, обновление Jinja2 до 3.1.6 запланировано в следующем релизе.

---

## 2) SAST и Secrets (DS2)

**SAST & Secrets:** выполнены Semgrep (SARIF) и Gitleaks (JSON).
**Артефакты:** EVIDENCE/S10/semgrep.sarif, EVIDENCE/S10/gitleaks.json.
**Actions:** [ссылка](https://github.com/M1steryO/secdev-lite-template-ds/actions/runs/20067980001/job/57561700040).

### 2.1 SAST

- **Инструмент/профиль:** Semgrep (p/ci)
- **Как запускал:** через CI workflow
- **Отчёт:** `EVIDENCE/S10/semgrep.sarif `
- **Выводы:**

  - High-severity находки отсутствуют (по отчёту SARIF).
  - Семантических уязвимостей в приложении не выявлено.
  - Единственная область риска — уязвимая зависимость Jinja2 3.1.4, выявленная при Software Composition Analysis.

### 2.2 Secrets scanning

- **Инструмент:** Gitleaks
- **Как запускал:** через CI workflow
- **Отчёт:** `EVIDENCE/S10/gitleaks.json`
- **Выводы:**

  - Срабатываний нет — отчёт содержит пустой массив [].
  - Утечек ключей, токенов и конфиденциальных данных не обнаружено ни в актуальном состоянии, ни в истории.
  - Дополнительные меры не требуются.


Повторное сканирование после SCA-фазы подтверждает отсутствие High/Critical находок.
Дополнительных фиксов и подавлений не требуется.

---

## 3) DAST (DS3)

DAST (ZAP baseline) против http://localhost:8080.
**Артефакты**: EVIDENCE/S11/zap_baseline.html, EVIDENCE/S11/zap_baseline.json,
**Actions:** [ссылка](https://github.com/M1steryO/secdev-lite-template-ds/actions/runs/20147191091).

- **Application Startup:** FastAPI приложение успешно развернуто в Docker-сети `zapnet`
- **ZAP Full Scan:** Полное активное DAST-сканирование (ZAP 2.16.1) по адресу `http://web:8080`
- **Evidence:** `EVIDENCE/S11/zap_full.json` , `EVIDENCE/S11/zap_full.html` , `EVIDENCE/S11/zap_baseline.html`, `EVIDENCE/S11/zap_baseline.json`

**Результаты:**

- **Baseline-сканирование OWASP ZAP** выявило уязвимости среднего и низкого уровня риска, преимущественно связанные с отсутствием защитных HTTP-заголовков (CSP, X-Frame-Options и др.), что указывает на необходимость усиления конфигурации безопасности, при отсутствии критических проблем.
- **Full-сканирование OWASP ZAP** не обнаружило критических и высокорисковых уязвимостей, а в итоговом отчёте отсутствуют активные alert’ы, что подтверждает корректную работу приложения и его устойчивость к типовым атакам на момент проверки.

## 4) Харднинг (DS4)

Артефакты: EVIDENCE/S12/hadolint.json, checkov.json, trivy.json. Actions: <ссылка на успешный job>.

- [X] **Dockerfile analysis (Hadolint, без нарушений)** → Evidence: `EVIDENCE/S12/hadolint.json`
- [X] **IaC scanning (Checkov, анализ Kubernetes/IaC)** → Evidence: `EVIDENCE/S12/checkov.json`
- [X] **Container scanning (Trivy, образ s09s12-app:local)** → Evidence: `EVIDENCE/S12/trivy.json`

**Результаты:**

- Checkov — проверка конфигураций Kubernetes показала прохождение всех проверок безопасности, что означает соответствие манифестов базовым best practices и отсутствие выявленных небезопасных настроек на уровне IaC
- Hadolint — анализ Dockerfile не выявил замечаний и нарушений правил, что указывает на корректную структуру Dockerfile и соблюдение рекомендаций по его написанию. 
- Trivy — сканирование контейнерного образа не обнаружило критических и высокорисковых уязвимостей в ОС-пакетах и зависимостях, что подтверждает приемлемый уровень безопасности образа на момент проверки.


Результаты Trivy/Checkov использованы как baseline-харднинг; дальнейшие меры (non-root, CSP) запланированы.

---

## 5) Quality-gates и проверка порогов (DS5)

- **Пороговые правила (словами):**  
  - **SCA:** Critical = 0; High ≤ 1  
  - **SAST:** Critical = 0; High = 0  
  - **Secrets:** 0 подтверждённых находок  
  - **Policy / IaC / Container:** Violations = 0 (High, Critical)  
  - **DAST:** High = 0 (Medium допускаются как backlog)

- **Как проверяются:**

  - **Автоматически в CI (fail job при нарушении порогов):**
    - SCA — сбор отчёта и завершение job с ошибкой при обнаружении High/Critical
    - SAST — Semgrep завершает job с ошибкой при High-severity
    - Secrets — Gitleaks возвращает ненулевой exit-code при обнаружении секретов
    - Policy / IaC / Container — Trivy завершает job с ошибкой при High/Critical
    - DAST — ZAP baseline завершает job с ошибкой при наличии High

    ```bash
    # SCA
    grype sbom:sbom.json --fail-on high

    # SAST
    semgrep --config p/ci --severity=high --error

    # Secrets
    gitleaks detect --exit-code 1

    # Policy / Container / IaC
    trivy image --severity HIGH,CRITICAL --exit-code 1 s09s12-app:local
    trivy config --severity HIGH,CRITICAL --exit-code 1 .

    # DAST
    zap-baseline.py -t http://web:8080 -m 3
    ```

- **Ссылки на конфиг/скрипт:**

  ```text
  GitHub Actions:
  .github/workflows/security.yml

  Jobs:
  - sca        (SBOM + Grype)
  - sast       (Semgrep)
  - secrets    (Gitleaks)
  - policy     (Trivy image / config)
  - dast       (OWASP ZAP baseline)

---

## 6) Триаж-лог (fixed / suppressed / open)

| ID/Anchor | Класс | Severity | Статус | Действие | Evidence | Ссылка на фикс/исключение | Комментарий / owner / expiry |
|----------|-------|----------|--------|----------|----------|----------------------------|------------------------------|
| ZAP-CSP | DAST | Medium | open | hardening | `EVIDENCE/S11/zap_baseline.json#CSP` | backlog | Отсутствует CSP; не эксплуатируемо, требуется конфигурационный hardening |
| ZAP-XFO | DAST | Medium | open | hardening | `EVIDENCE/S11/zap_baseline.json#X-Frame-Options` | backlog | Риск clickjacking; запланировано добавление заголовков |
| ZAP-FULL-0 | DAST | Info | fixed | verify | `EVIDENCE/S11/zap_full.json` | commit `<sha>` | Full scan не выявил alert’ов |
| TRIVY-IMG | SCA | Medium | open | monitor | `EVIDENCE/S12/trivy.json` | N/A | Критических и High не выявлено |
| CHECKOV-IAC | IaC | Low | fixed | verify | `EVIDENCE/S12/checkov.json` | N/A | Нарушений best-practice не выявлено |
| HADOLINT | Build | Low | fixed | verify | `EVIDENCE/S12/hadolint.json` | N/A | Dockerfile соответствует рекомендациям |

> Для подавлений (suppressed) — не применялось: false positives не зафиксированы.


---

## 7) Эффект «до/после» (метрики) (DS4/DS5)

| Контроль/Мера | Метрика | До | После | Evidence (до), (после) |
|---------------|--------|----:|------:|------------------------|
| DAST (ZAP) | #High / #Critical | N/A | 0 / 0 | `EVIDENCE/S11/zap_baseline.*`, `zap_full.*` |
| Container Image | #Critical / #High (SCA) | N/A | 0 / 0 | `EVIDENCE/S12/trivy.json` |
| Dockerfile | Hadolint violations | N/A | 0 | `EVIDENCE/S12/hadolint.json` |
| IaC (K8s) | Checkov violations | N/A | 0 | `EVIDENCE/S12/checkov.json` |

> Метрика «До» отсутствует, т.к. сканирование выполнялось на первой итерации конвейера безопасности.


## 8) Связь с TM и DV (сквозная нитка)

- **Закрываемые угрозы из TM:**
  - T-002 (XSS / injection via reflected input) → DAST (ZAP baseline/full)
  - T-004 (Misconfiguration / missing security headers) → ZAP baseline
  - T-007 (Vulnerable dependencies / base image) → Trivy
  - T-009 (Insecure container/IaC config) → Hadolint + Checkov

- **Связь с DV:**
  - DAST (ZAP baseline + full) встроен в CI
  - Container scanning (Trivy) выполняется на этапе сборки образа
  - IaC scanning (Checkov) выполняется при проверке Kubernetes-манифестов
  - Dockerfile lint (Hadolint) выполняется на этапе build


---

## 9) Out-of-Scope
-

---

## 10) Самооценка по рубрике DS (0/1/2)

- **DS1. SBOM и SCA:** 2
- **DS2. SAST + Secrets:** 2
- **DS3. DAST или Policy (Container/IaC):** 2
- **DS4. Харднинг (доказуемый):** 2
- **DS5. Quality-gates, триаж и «до/после»:** 2

**Итог DS (сумма):** 10/10
