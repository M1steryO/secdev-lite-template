# DV - Мини-проект «DevOps-конвейер»

> Этот файл - **индивидуальный**. Его проверяют по **rubric_DV.md** (5 критериев × {0/1/2} → 0-10).
> Подсказки помечены `TODO:` - удалите после заполнения.
> Все доказательства/скрины кладите в **EVIDENCE/** и ссылайтесь на конкретные файлы/якоря.

---

## 0) Мета

- **Проект:** учебный шаблон secdev-lite-template-dv
- **Версия:** 2025/12/01
- **Кратко:** воспроизводимая сборка проекта, применение патчей безопасности (защита от SQL-инъекций, XSS и неверного ввода), а также автоматический прогон 5 тестов с выгрузкой отчёта в EVIDENCE/S06/test-report.xml.

---

## 1) Воспроизводимость локальной сборки и тестов (DV1)

- **Одна команда для сборки/тестов:**

  ```bash
  # ОС: macOS 14.5 / zsh (Darwin)
  # Python: 3.12.3
  make ci-2
  ```

  _Если без Makefile: укажите последовательность команд._
- **Версии инструментов (фиксация):**

Конкретные версии Python-библиотек заданы в `requirements.txt`.Для полного воспроизведения окружения их фактический снимок также сохранён в артефакте [`pip-freeze.txt`](../EVIDENCE/S06/pip-freeze.txt).

- **Описание шагов (кратко):**

1. Склонировать репозиторий

```bash
  git clone https://github.com/M1steryO/secdev-lite-template-dv.git 
```

2. Перейти в папку проекта:

```bash
  cd secdev-lite-template-dv
```

3. Запустить проект:

```bash
  make ci-2
```

4. Просмотреть отчёт в:
   `EVIDENCE/S06/test-report.xml`.

## 2) Контейнеризация (DV2)

- **Dockerfile:** `./Dockerfile`

  - Базовый образ: `python:3.11-slim` (минимальный образ, single-stage).
  - На уровне образа:
    - выставлены флаги `PYTHONDONTWRITEBYTECODE=1`, `PYTHONUNBUFFERED=1`, `PIP_DISABLE_PIP_VERSION_CHECK=1`;
    - ставятся зависимости из `requirements.txt` через `pip install --no-cache-dir -r requirements.txt`;
    - копируются каталоги `app/` и `scripts/`;
    - создаётся непривилегированный пользователь `appuser` (`UID=10001`), ему передаётся владение `/app` (`chown -R appuser:appuser /app`);
    - контейнер запускается под пользователем `USER appuser`;
    - добавлен `HEALTHCHECK` (каждые 10s, timeout 3s, 5 попыток) с проверкой `http://127.0.0.1:8000/` через встроенный Python;
    - `EXPOSE 8000`;
    - `CMD`: `python scripts/init_db.py && uvicorn app.main:app --host 0.0.0.0 --port 8000`.
- **Сборка/запуск локально:**

  ```bash
  docker build -t secdev-seed:local .
  docker run --rm -p 8000:8000 secdev-seed:local
  ```

  Приложение поднимается на `http://localhost:8000/`, внутри контейнера слушает `0.0.0.0:8000`. Healthcheck из Dockerfile ходит на `http://127.0.0.1:8000/` внутри контейнера.
- **docker-compose:** `./docker-compose.yml`
  Сервис `web`:

  * `build: .`
  * `image: secdev-seed:latest`
  * `restart: unless-stopped`
  * `ports: ["8000:8000"]`
  * `environment: APP_NAME=secdev-seed, DEBUG=true`
  * `healthcheck`: `curl -f http://localhost:8000/` с интервалом 10s, timeout 3s, 5 ретраев.

---

## 3) CI: базовый pipeline и стабильный прогон (DV3)

- **Платформа CI:**  GitHub Actions
- **Файл конфига CI:**  `.github/workflows/ci.yml`
- **Стадии (минимум):** checkout → setup-python → install-deps → init db → **test** → **docker-build**
- **Фрагмент конфигурации (ключевые шаги):**

  ```yaml
  jobs:
    build-test:
      runs-on: ubuntu-latest

      steps:
        - name: Checkout
          uses: actions/checkout@v4

        - name: Setup Python
          uses: actions/setup-python@v5
          with:
            python-version: '3.11'

        - name: Cache pip
          uses: actions/cache@v4
          with:
            path: ~/.cache/pip
            key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
            restore-keys: |
              ${{ runner.os }}-pip-

        - name: Install deps
          run: pip install -r requirements.txt

        - name: Init DB
          run: python scripts/init_db.py

        - name: Run tests
          run: |
            mkdir -p EVIDENCE/S08
            pytest -q --junitxml=EVIDENCE/S08/test-report.xml

        - name: Upload artifacts
          if: always()
          uses: actions/upload-artifact@v4
          with:
            name: evidence-s08
            path: EVIDENCE/S08/**
            if-no-files-found: warn

  ```
- **Стабильность:**

  Все последние запуски workflow  **успешные**. После добавления пайплайна и фикса тестов раннер стабильно отрабатывает.
- **Ссылка/копия лога прогона:** `EVIDENCE/S08/test-report.xml`

---

## 4) Артефакты и логи конвейера (DV4)

_Сложите файлы в `/EVIDENCE/` и подпишите их назначение._

| Артефакт/лог                      | Путь в `EVIDENCE/` | Комментарий                                                                                                   |
| -------------------------------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Отчет о тестах                   | `S06/test-report.xml`   | JUnit XML отчет, подтверждающий успешное прохождение 5/5 тестов              |
| Лог локального запуска   | `S06/local-run-log.txt`    | Лог выполнения команды `make ci-2`.                                                                |
| Freeze/версии инструментов | `S06/pip-freeze.txt`   | Список точных версий всех зависимостей для 100% воспроизводимости. |

---

## 5) Секреты и переменные окружения (DV5 - гигиена, без сканеров)

- **Шаблон окружения:** добавлен файл `.env.example` со списком переменных (без значений):

  - `APP_NAME=secdev-seed`
  - `DEBUG=true`
  - `SECRET_KEY=change-me-in-real-projects`
- **Хранение и передача в CI:**

  - Секреты лежат в настройках репозитория (**masked**),
  - В pipeline они **не печатаются** в явном виде.
  - Логи **не содержат** чувствительной информации
- **Автоматическая проверка утечек секретов:**

  ```yaml
  -- name: Secrets Scan
  run: |
    docker run --rm \
      -v "$(pwd)":/code \
      zricethezav/gitleaks:latest detect \
      --source="/code" \
      --report-format=json \
      --report-path=/code/EVIDENCE/S10/gitleaks.json
  ```
- **Политика ротации секретов:**

  - При выявлении утечки выполняется немедленное уведомление и блокировка скомпрометированного ключа
  - Обновление секретов производится через интерфейс GitHub Actions Secrets
  - После обновления выполняется повторное сканирование кода, чтобы убедиться в отсутствии остатков старых ключей

---

## 6) Индекс артефактов DV

_Чтобы преподаватель быстро сверил файлы._

| Тип                            | Файл в `EVIDENCE/`        |
|--------------------------------|---------------------------|
| Логи локального запуска тестов | `S06/local-run-log.txt`     |
| Зависимости pip                | `S06/pip-freeze.txt`      |
| Репорт тестов                  | `S06/test-report.xml`     |
| Лог билда докера               | `S07/build.log`           |
| Лог билда docker-compose       | `S07/compose-up.log`      |
| Размер контейнера              | `S07/image-size.txt`      |
| Код ответа главной страницы    | `S07/http_root_code.json` |
| Health                         | `S07/health.json`         |
| Inspect Web                    | `S07/inspect_web.json`    |
| Non-root                       | `S07/non-root.txt`        |
| Лог запуска контейнера         | `S07/run.log`             |
| Лог CI                         | `S08/ci-log.txt`          |
| Ссылка на последний запуск CI  | `S08/ci-run.txt`          |
| Покрытие тестами               | `S08/coverage.xml`        |
| Отчёт по тестам                | `S08/test-report.xml`     |

---

## 7) Связь с TM и DS (hook)

- **TM:** этот конвейер обслуживает риски процесса сборки/поставки (например, культура работы с секретами, воспроизводимость).
- **DS:** сканы/гейты/триаж будут оформлены в `DS.md` с артефактами в `EVIDENCE/`.

---

## 8) Самооценка по рубрике DV (0/1/2)

- **DV1. Воспроизводимость локальной сборки и тестов:** [ ] 0 [ ] 1 [ ] 2
- **DV2. Контейнеризация (Docker/Compose):** [ ] 0 [ ] 1 [ ] 2
- **DV3. CI: базовый pipeline и стабильный прогон:** [ ] 0 [ ] 1 [ ] 2
- **DV4. Артефакты и логи конвейера:** [ ] 0 [ ] 1 [ ] 2
- **DV5. Секреты и конфигурация окружения (гигиена):** [ ] 0 [ ] 1 [ ] 2

**Итог DV (сумма):** __/10
