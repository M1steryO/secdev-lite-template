# DS - Отчёт «DevSecOps-сканы и харднинг»

> Этот файл - **индивидуальный**. Его проверяют по **rubric_DS.md** (5 критериев × {0/1/2} → 0-10).
> Подсказки помечены `TODO:` - удалите после заполнения.
> Все доказательства/скрины кладите в **EVIDENCE/** и ссылайтесь на конкретные файлы/якоря.

---

## 0) Мета

- **Проект:** учебный шаблон secdev-lite-template-ds
- **Версия:** 2025/12/09

---

## 1) SBOM и уязвимости зависимостей (DS1)

**SBOM & SCA**: сгенерирован SBOM (CycloneDX), выполнен SCA.
**Артефакты**: EVIDENCE/S09/sbom.json, EVIDENCE/S09/sca_report.json.
**Сводка**: EVIDENCE/S09/sca_summary.md.
**Actions**: [ссылка](https://github.com/M1steryO/secdev-lite-template-ds/actions/runs/20066055689/job/57554790061).

В пакете Jinja2 обнаружены **3 уязвимости** среднего и высокого уровня, связанные с обходом sandbox: **CVE-2024-56326, CVE-2025-27516, CVE-2024-56201**.

**Рекомендуемое действие:** обновить Jinja2 до версии  **3.1.6 или выше** .

---

## 2) SAST и Secrets (DS2)

**SAST & Secrets:** выполнены Semgrep (SARIF) и Gitleaks (JSON).
**Артефакты:** EVIDENCE/S10/semgrep.sarif, EVIDENCE/S10/gitleaks.json.
**Actions:** [ссылка](https://github.com/M1steryO/secdev-lite-template-ds/actions/runs/20067980001/job/57561700040).

### 2.1 SAST

- **Инструмент/профиль:** Semgrep (p/ci)
- **Как запускал:**

  ```bash
  semgrep --config p/ci --severity=high --error --json --output EVIDENCE/sast-YYYY-MM-DD.json
  ```
- **Отчёт:** `EVIDENCE/S10/semgrep.sarif `
- **Выводы:**

  - High-severity находки отсутствуют (по отчёту SARIF).
  - Семантических уязвимостей в приложении не выявлено.
  - Единственная область риска — уязвимая зависимость Jinja2 3.1.4, выявленная при Software Composition Analysis.

### 2.2 Secrets scanning

- **Инструмент:** Gitleaks
- **Как запускал:**

  ```bash
  gitleaks detect --no-git --report-format json --report-path EVIDENCE/secrets-YYYY-MM-DD.json
  gitleaks detect --log-opts="--all" --report-format json --report-path EVIDENCE/secrets-YYYY-MM-DD-history.json
  ```
- **Отчёт:** `EVIDENCE/S10/gitleaks.json`
- **Выводы:**

  - Срабатываний нет — отчёт содержит пустой массив [].
  - Утечек ключей, токенов и конфиденциальных данных не обнаружено ни в актуальном состоянии, ни в истории.
  - Дополнительные меры не требуются.

---

## 3) DAST **или** Policy (Container/IaC) (DS3)

DAST (ZAP baseline) против http://localhost:8080.
**Артефакты**: EVIDENCE/S11/zap_baseline.html, EVIDENCE/S11/zap_baseline.json.
**Actions:** [ссылка](https://github.com/M1steryO/secdev-lite-template-ds/actions/runs/20067979889/job/57561699938).

- **Application Startup:** FastAPI приложение успешно развернуто в Docker-сети `zapnet`
- **ZAP Full Scan:** Полное активное DAST-сканирование (ZAP 2.16.1) по адресу `http://web:8080`
- **Evidence:** `EVIDENCE/S11/zap_full.json` / `zap_full.html`

**Результаты:**

- 0 выявленных уязвимостей (High/Medium/Low)
- 0 информационных предупреждений
- 0 ложных срабатывани й

Сканирование прошло **успешно**: ни один тест безопасности не выявил рисков

## 4) Харднинг (доказуемый) (DS4)

Отметьте **реально применённые** меры, приложите доказательства из `EVIDENCE/`.

- [ ] **Контейнер non-root / drop capabilities** → Evidence: `EVIDENCE/policy-YYYY-MM-DD.txt#no-root`
- [ ] **Rate-limit / timeouts / retry budget** → Evidence: `EVIDENCE/load-after.png`
- [ ] **Input validation** (типы/длины/allowlist) → Evidence: `EVIDENCE/sast-YYYY-MM-DD.*#input`
- [ ] **Secrets handling** (нет секретов в git; хранилище секретов) → Evidence: `EVIDENCE/secrets-YYYY-MM-DD.*`
- [ ] **HTTP security headers / CSP / HTTPS-only** → Evidence: `EVIDENCE/security-headers.txt`
- [ ] **AuthZ / RLS / tenant isolation** → Evidence: `EVIDENCE/rls-policy.txt`
- [ ] **Container/IaC best-practice** (минимальная база, readonly fs, …) → Evidence: `EVIDENCE/trivy-YYYY-MM-DD.txt#cfg`

> Для «1» достаточно ≥2 уместных мер с доказательствами; для «2» - ≥3 и хотя бы по одной показать эффект «до/после».

---

## 5) Quality-gates и проверка порогов (DS5)

- **Пороговые правила (словами):**Примеры: «SCA: Critical=0; High≤1», «SAST: Critical=0», «Secrets: 0 истинных находок», «Policy: Violations=0».
- **Как проверяются:**

  - Ручной просмотр (какие файлы/строки) **или**
  - Автоматически:  (скрипт/job, условие fail при нарушении)

    ```bash
    SCA: grype ... --fail-on high
    SAST: semgrep --config p/ci --severity=high --error
    Secrets: gitleaks detect --exit-code 1
    Policy/IaC: trivy (image|config) --severity HIGH,CRITICAL --exit-code 1
    DAST: zap-baseline.py -m 3 (фейл при High)
    ```
- **Ссылки на конфиг/скрипт (если есть):**

  ```bash
  GitHub Actions: .github/workflows/security.yml (jobs: sca, sast, secrets, policy, dast)
  или GitLab CI: .gitlab-ci.yml (stages: security; jobs: sca/sast/secrets/policy/dast)
  ```

---

## 6) Триаж-лог (fixed / suppressed / open)

| ID/Anchor     | Класс | Severity | Статус | Действие | Evidence                              | Ссылка на фикс/исключение | Комментарий / owner / expiry |
| ------------- | ---------- | -------- | ------------ | ---------------- | ------------------------------------- | ----------------------------------------------- | --------------------------------------- |
| CVE-2024-XXXX | SCA        | High     | fixed        | bump             | `EVIDENCE/deps-YYYY-MM-DD.json#CVE` | `commit abc123`                               | -                                       |
| ZAP-123       | DAST       | Medium   | suppressed   | ignore           | `EVIDENCE/dast-YYYY-MM-DD.pdf#123`  | `EVIDENCE/suppressions.yml#zap`               | FP; owner: ФИО; expiry: 2025-12-31   |
| SAST-77       | SAST       | High     | open         | backlog          | `EVIDENCE/sast-YYYY-MM-DD.*#77`     | issue-link                                      | план фикса в релизе N   |

> Для «2» по DS5 обязательно указывать **owner/expiry/обоснование** для подавлений.

---

## 7) Эффект «до/после» (метрики) (DS4/DS5)

| Контроль/Мера | Метрика                  | До | После | Evidence (до), (после)                          |
| ------------------------- | ------------------------------- | ---: | ---------: | ------------------------------------------------------ |
| Зависимости    | #Critical / #High (SCA)         | TODO |    0 / ≤1 | `EVIDENCE/deps-before.json`, `deps-after.json`     |
| SAST                      | #Critical / #High               | TODO |    0 / ≤1 | `EVIDENCE/sast-before.*`, `sast-after.*`           |
| Secrets                   | Истинные находки | TODO |          0 | `EVIDENCE/secrets-*.json`                            |
| Policy/IaC                | Violations                      | TODO |          0 | `EVIDENCE/checkov-before.txt`, `checkov-after.txt` |

---

## 8) Связь с TM и DV (сквозная нитка)

- **Закрываемые угрозы из TM:** TODO: T-001, T-005, … (ссылки на таблицу трассировки TM)
- **Связь с DV:** TODO: какие сканы/проверки встроены или будут встраиваться в pipeline

---

## 9) Out-of-Scope

- TODO: что сознательно не сканировалось сейчас и почему (1-3 пункта)

---

## 10) Самооценка по рубрике DS (0/1/2)

- **DS1. SBOM и SCA:** [ ] 0 [ ] 1 [ ] 2
- **DS2. SAST + Secrets:** [ ] 0 [ ] 1 [ ] 2
- **DS3. DAST или Policy (Container/IaC):** [ ] 0 [ ] 1 [ ] 2
- **DS4. Харднинг (доказуемый):** [ ] 0 [ ] 1 [ ] 2
- **DS5. Quality-gates, триаж и «до/после»:** [ ] 0 [ ] 1 [ ] 2

**Итог DS (сумма):** __/10
