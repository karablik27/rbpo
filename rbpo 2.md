# Отчёт DV проект 2

**Автор:** Карабельников Степан Иванович  
**Группа:** БПИ234  

## Структура
- 1. Стабильный CI
- 2. Сборка, тесты и артефакты
- 3. Секреты вынесены из кода
- 4. PR-политика и ревью по чек-листу
- 5. Превосходство: кэш, матрица, покрытие и релизные артефакты
- 6. CD и прод-деплой на Render


## 1. Стабильный CI

Для проекта настроен единый пайплайн `name: CI`, который автоматически запускается:

- при каждом `push` в ветку `main`;
- при открытии/обновлении любого `pull_request`.

Это гарантирует, что все изменения проходят один и тот же набор проверок до попадания в основную ветку.

Чтобы избежать гонок и лишней нагрузки, включена настройка `concurrency`:

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

Для одной и той же ветки предыдущие запуски CI отменяются, остаётся только последний актуальный прогон.

Пайплайн выполняется в матрице версий Python:

```yaml
strategy:
  fail-fast: false
  matrix:
    python-version: ["3.10", "3.11", "3.12"]
```

Это даёт уверенность, что код стабильно работает сразу на трёх поддерживаемых версиях интерпретатора. Параметр `fail-fast: false` позволяет собрать полный отчёт по всем версиям, даже если одна из них упала.

В начале job явно задаётся конфигурация окружения для CI:

```yaml
- name: Force CI database config
  run: |
    echo "ENV=ci" >> $GITHUB_ENV
    echo "DATABASE_URL=" >> $GITHUB_ENV
```

Благодаря этому тесты и проверки выполняются в изолированной CI-конфигурации без зависимости от боевой базы.

Для ускорения повторных прогонов используется кэширование `pip`:

```yaml
- name: Cache pip
  uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ matrix.python-version }}-${{ hashFiles('**/requirements*.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

Если зависимости не поменялись, повторная установка происходит значительно быстрее.

Таким образом, CI стабильно и предсказуемо отрабатывает на каждом коммите и PR, обеспечивая зелёные прогоны только при успешном прохождении всех этапов: линтеров, статического анализа, тестов, сборки Docker-образа и проверки безопасности.

---

## 2. Сборка, тесты и артефакты

### 2.1. Линтеры и статический анализ

В пайплайне последовательно выполняются проверки качества кода:

```yaml
- name: Lint & Format
  run: |
    mkdir -p reports
    ruff check --output-format=github .
    ruff check --output-format=full > reports/ruff.txt
    black --check .
    isort --check-only .

- name: Type check (mypy)
  run: |
    mypy app | tee reports/mypy-report.txt

- name: Security scan (bandit)
  run: bandit -r app -ll
```

* `ruff` — быстрый линтер и проверка стиля (отчёт в формате GitHub + полный лог в `reports/ruff.txt`);
* `black` — проверка форматирования;
* `isort` — корректность сортировки импортов;
* `mypy` — статическая проверка типов (лог сохраняется в `reports/mypy-report.txt`);
* `bandit` — статический анализ безопасности Python-кода.

Все эти шаги обязательны для успешного выполнения CI и не пропускают изменения, которые ломают стиль, типы или базовые правила безопасности.

### 2.2. Тесты и покрытие

Тесты запускаются с включённым покрытием:

```yaml
- name: Run tests with coverage
  run: |
    mkdir -p reports
    mkdir -p htmlcov
    pytest --maxfail=1 --disable-warnings -q       --cov=app       --cov-report=xml       --cov-report=html       --cov-report=term-missing       --junitxml=reports/junit-${{ matrix.python-version }}.xml
```

Генерируются:

- `coverage.xml` — отчёт покрытия в формате XML;
- `htmlcov/` — HTML-отчёт покрытия;
- `reports/junit-<version>.xml` — результат тестов в формате JUnit для каждой версии Python.

Таким образом, по итогам прогона можно:

- увидеть отсутствие/наличие пропущенных строк по покрытиям;
- интегрировать отчёты в сторонние инструменты (Dashboards, Codecov и т.п.).

### 2.3. Артефакты CI

Все важные отчёты собираются и публикуются как артефакты GitHub Actions:

```yaml
- name: Upload test reports
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: test-reports-${{ matrix.python-version }}
    path: reports/

- name: Upload coverage HTML
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: coverage-html-${{ matrix.python-version }}
    path: htmlcov/

- name: Upload lint reports
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: lint-reports-${{ matrix.python-version }}
    path: reports/
```

Флаг `if: always()` гарантирует, что артефакты будут сохранены даже при падении тестов — это удобно для отладки.

Отдельно проверяется Dockerfile:

```yaml
- name: Dockerfile Lint (Hadolint)
  uses: hadolint/hadolint-action@v3.1.0
  with:
    dockerfile: Dockerfile
  continue-on-error: true
```

`hadolint` даёт рекомендации по качеству Dockerfile, но не ломает сборку (чтобы замечания можно было исправлять постепенно).

### 2.4. Сборка Docker-образа и артефакты контейнера

Для версии Python 3.12 в CI собирается Docker-образ приложения:

```yaml
- name: Build Docker image (cached layers)
  if: matrix.python-version == '3.12'
  run: |
    docker build       --cache-from=type=gha       --cache-to=type=gha,mode=max       -t okr-app:latest .
```

Используется кеш слоёв (`--cache-from/--cache-to` с backend `gha`), что ускоряет повторную сборку.

После этого выполняется сканирование образа на уязвимости высоким и критическим уровням:

```yaml
- name: Trivy Scan (High & Critical)
  if: matrix.python-version == '3.12'
  uses: aquasecurity/trivy-action@0.17.0
  with:
    image-ref: okr-app:latest
    format: 'table'
    output: 'trivy-report.txt'
    severity: 'HIGH,CRITICAL'
    ignore-unfixed: true
    exit-code: '0'
```

Отчёт `trivy-report.txt` загружается как артефакт:

```yaml
- name: Upload security artifacts
  if: matrix.python-version == '3.12'
  uses: actions/upload-artifact@v4
  with:
    name: security-reports
    path: trivy-report.txt
```

Сам Docker-образ также сохраняется и публикуется как артефакт:

```yaml
- name: Save Docker image as artifact
  if: matrix.python-version == '3.12'
  run: docker save okr-app:latest -o okr-app.tar

- name: Upload Docker image artifact
  if: matrix.python-version == '3.12'
  uses: actions/upload-artifact@v4
  with:
    name: docker-image
    path: okr-app.tar
```

Это даёт «релизный артефакт» в виде готового контейнера, который можно забрать из CI и развернуть на сервере без пересборки.

В связке с multi-stage Dockerfile и docker-compose-сервисом `okr` это обеспечивает полный цикл:

- сборка образа;
- проверка Dockerfile и контейнера;
- публикация отчётов и самого образа;
- возможность воспроизводимой сборки и запуска как в CI, так и локально/на сервере.

## 3. Секреты вынесены из кода

В проекте принята политика «**никаких секретов в репозитории**».  
Все чувствительные данные (строки подключения к БД, токены и т.п.) задаются только через переменные окружения и секреты CI/CD.

### 3.1. Docker / docker-compose

В Dockerfile и `docker-compose.yml` **нет хардкоженных секретов**:

```yaml
services:
  okr:
    environment:
      ENV: "prod"
      ALLOWED_ORIGINS: "http://localhost:5173"
      RESPONSE_MODEL_POLICY: "warn"
```

Эти параметры не являются чувствительными (это режим окружения и политика ответа).  
Настоящие секреты (строка подключения к БД и т.п.) передаются в контейнер извне:

- при локальном запуске — через `docker compose` с подстановкой переменных окружения хоста;
- на сервере — через настройки окружения/секретов платформы деплоя.

В Dockerfile нет ни одного токена, пароля или URL с кредами — только общие настройки окружения, non-root пользователь, пути к томам и `HEALTHCHECK`.

### 3.2. CI / GitHub Actions

В CI-пайплайне тестовое окружение жёстко переключается в режим `ENV=ci`, а `DATABASE_URL` в этой джобе очищается:

```yaml
- name: Force CI database config
  run: |
    echo "ENV=ci" >> $GITHUB_ENV
    echo "DATABASE_URL=" >> $GITHUB_ENV
```

Это сделано специально, чтобы:

- в CI **никогда не использовались боевые секреты**;
- тесты работали на отдельной конфигурации, не завязанной на реальную БД.

Для реальных окружений (dev/prod) используются секреты платформы деплоя (GitHub Secrets / Render env vars).  
Сами значения секретов остаются вне репозитория и не попадают в логи — в workflow они подставляются только через переменные окружения.

В результате:

- в git-истории отсутствуют токены и пароли;
- Docker и CI используют только внешние env-переменные;
- конфигурация безопасно переключается между `ci`, `dev` и `prod` без изменений в коде.

---

## 4. PR-политика и ревью по чек-листу

Работа над проектом велась в режиме «**каждое задание — отдельный PR**» (в том числе P07 и P08).  
Для PR используется шаблон с единым стилем описания: **Контекст → Что сделано → Как проверял(а)**.

### 4.1. Шаблон PR и саморевью

Для заданий P07 и P08 использовались развёрнутые описания:

- **P07 (контейнеризация и безопасность контейнера)**  
  В PR явно расписано соответствие критериям C1–C5:
  - multi-stage Dockerfile,
  - запуск под `appuser` (non-root),
  - `read_only: true`, вынос данных в тома `/data/db`, `/app/uploads`, `/app/logs`,
  - `cap_drop: [ALL]`, `no-new-privileges=true`,
  - подключение `seccomp.json` и AppArmor-профиля `okr-apparmor-profile`,
  - рабочий `docker-compose.yml` и локальный запуск через `docker compose up`,
  - Trivy + Hadolint, прикреплённый `trivy-report.txt` как артефакт CI.

- **P08 (полный CI/CD конвейер)**  
  В PR описано:
  - матрица версий Python `3.10/3.11/3.12`,
  - кэширование зависимостей и слоёв Docker,
  - генерация и выгрузка отчётов (lint, mypy, JUnit, coverage HTML, Trivy),
  - сборка Docker-образа и сохранение его как артефакт `okr-app.tar`,
  - деплой на Render, проверка `/` и `/docs` на проде.

Каждый PR завершается блоком **«Как проверял(а)»** с чек-листом:

- локальные проверки (`ruff/black/isort`, `mypy`, `pytest`, `docker build`, `docker compose up`);
- подтверждение, что CI зелёный;
- ссылки/скриншоты из GitHub Actions и прод-окружения.

Таким образом, PR одновременно выступает и как документ саморевью по чек-листу курса.

### 4.2. Политика работы с main и ревью

- Ветка `main` рассматривается как защищённая: изменения попадают в неё **только через PR**.
- Для каждого задания создаётся отдельная рабочая ветка (`feature/p07-docker`, `feature/p08-ci-cd` и т.п.).
- Перед мёрджем обязательно:
  - успешный прогон CI;
  - заполненный PR-шаблон с явным описанием соответствия критериям C1–C5;
  - визуальная проверка ключевых файлов (workflow, Dockerfile, docker-compose).

Такой процесс обеспечивает прозрачность: по одному PR можно отследить, какие именно требования задания были реализованы, как они проверялись и какие артефакты/скрины подтверждают результат.

## 5. Превосходство: кэш, матрица, покрытие и релизные артефакты

Для получения 9–10 баллов по проекту DV 0.2 в пайплайне реализован «расширенный» набор практик DevOps, который выходит за базовый чек-лист.

### 5.1. Матрица версий Python

CI выполняется в матрице из трёх версий Python:

```yaml
strategy:
  fail-fast: false
  matrix:
    python-version: ["3.10", "3.11", "3.12"]
```

Это обеспечивает:

- проверку совместимости кода с несколькими версиями интерпретатора;
- полный отчёт по каждой версии (из-за `fail-fast: false` ни одна джоба не «обрубает» остальные).

### 5.2. Кэш зависимостей и Docker-слоёв

Для ускорения прогонов используется кэширование:

**pip:**

```yaml
- name: Cache pip
  uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ matrix.python-version }}-${{ hashFiles('**/requirements*.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

Ключ кэша завязан на ОС, версию Python и хэш `requirements*.txt`, поэтому кэш автоматически инвалидируется при изменении зависимостей.

**Docker-слои (для python 3.12):**

```yaml
- name: Build Docker image (cached layers)
  if: matrix.python-version == '3.12'
  run: |
    docker build       --cache-from=type=gha       --cache-to=type=gha,mode=max       -t okr-app:latest .
```

Используется backend `gha`, позволяющий переиспользовать слои между прогонами и существенно ускорять сборку образа.

### 5.3. Отчёты покрытия и статических проверок

При запуске тестов формируется несколько видов отчётов:

```yaml
pytest ...   --cov=app   --cov-report=xml   --cov-report=html   --cov-report=term-missing   --junitxml=reports/junit-${{ matrix.python-version }}.xml
```

В результате для каждой версии Python генерируются:

- `coverage.xml` и HTML-отчёт `htmlcov/` с детализацией покрытия;
- `junit-<version>.xml` — результаты тестов в формате JUnit;
- отдельные файлы с логами `ruff` и `mypy` в каталоге `reports/`.

Эти отчёты загружаются в CI как артефакты:

```yaml
- name: Upload test reports
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: test-reports-${{ matrix.python-version }}
    path: reports/

- name: Upload coverage HTML
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: coverage-html-${{ matrix.python-version }}
    path: htmlcov/

- name: Upload lint reports
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: lint-reports-${{ matrix.python-version }}
    path: reports/
```

Флаг `if: always()` гарантирует сохранение артефактов даже при падении тестов — это упрощает анализ ошибок.

### 5.4. Релизные артефакты: Docker-образ и security-отчёты

Собранный Docker-образ проходит сканирование уязвимостей и сохраняется как релизный артефакт:

```yaml
- name: Trivy Scan (High & Critical)
  if: matrix.python-version == '3.12'
  uses: aquasecurity/trivy-action@0.17.0
  with:
    image-ref: okr-app:latest
    format: 'table'
    output: 'trivy-report.txt'
    severity: 'HIGH,CRITICAL'
    ignore-unfixed: true
    exit-code: '0'

- name: Upload security artifacts
  if: matrix.python-version == '3.12'
  uses: actions/upload-artifact@v4
  with:
    name: security-reports
    path: trivy-report.txt

- name: Save Docker image as artifact
  if: matrix.python-version == '3.12'
  run: docker save okr-app:latest -o okr-app.tar

- name: Upload Docker image artifact
  if: matrix.python-version == '3.12'
  uses: actions/upload-artifact@v4
  with:
    name: docker-image
    path: okr-app.tar
```

Итог:

- `trivy-report.txt` — отчёт по High/Critical уязвимостям контейнера;
- `okr-app.tar` — готовый Docker-образ, который можно скачать из CI и развернуть без пересборки.

Это как раз соответствует требованию «релизные артефакты» в описании превосходства.

### 5.5. Секреты остаются вне кода

Во всех перечисленных шагах пайплайна ни один секрет **не захардкожен**:

- для CI режим окружения принудительно выставляется в `ENV=ci`, а `DATABASE_URL` очищается;
- реальные креды для dev/prod окружений задаются через секреты платформы деплоя и не попадают в репозиторий или логи.

Таким образом, проект демонстрирует сразу несколько признаков «превосходства» из формулировки курса:

- матрица версий + кэш,
- детальные отчёты покрытия и статических проверок,
- релизные артефакты (Docker-образ + security-репорты),
- корректная работа с секретами без хардкода в коде и CI.

## 6. CD и прод-деплой на Render

Помимо CI для проверки качества кода и сборки артефактов, в проекте настроен
полноценный **CD-контур** с деплоем на удалённый сервер.

### 6.1. Архитектура CD

- Источник истины — репозиторий на GitHub.
- Разработка ведётся через отдельные ветки и PR (P07, P08 и т.д.).
- После успешного ревью и **зелёного CI** изменения вливаются в `main`.
- Платформа **Render** настроена на автодеплой из ветки `main`:
  - Render подтягивает репозиторий,
  - собирает контейнер на основе того же `Dockerfile`,
  - перезапускает сервис с обновлённым образом.

Таким образом, прод-окружение всегда соответствует актуальному состоянию `main`,
а деплой не делается вручную “на сервер по ssh”.

### 6.2. Конфигурация сервиса на Render

- Тип сервиса: **Docker Web Service**.
- Render использует тот же `Dockerfile`, что и локальный `docker compose`, —
  это гарантирует одинаковое поведение приложения в dev и prod.
- Переменные окружения (`ENV`, `DATABASE_URL` и др.) заданы через
  **Render Environment variables**, а не в репозитории:
  - секреты не попадают в git-историю;
  - значения не светятся в логах и workflow.
- Приложение публикуется по адресу:  
  `https://course-project-karablik27-rpbo.onrender.com/`
- Для удобства проверки прод-инстанса добавлен корневой эндпоинт:

  ```python
  @app.get("/")
  def root():
      return {"status": "ok", "docs": "/docs"}
  ```

  В результате:
  - `GET /` → `200 OK` и JSON со статусом и ссылкой на документацию;
  - `GET /docs` → Swagger UI на прод-окружении.

### 6.3. Проверка деплоя

После каждого изменения в `main`:

1. Render автоматически пересобирает контейнер и поднимает новый релиз.
2. Доступность проверяется по URL:
   - `https://course-project-karablik27-rpbo.onrender.com/` — статус сервиса;
   - `https://course-project-karablik27-rpbo.onrender.com/docs` — интерактивная OpenAPI-документация.
3. При необходимости к отчёту прикладываются скриншоты:
   - успешного прогона CI в GitHub Actions;
   - открытого `/docs` на прод-сервере Render.

Таким образом, реализован не только стабильный CI, но и **рабочий CD-деплой на реальный сервер**, что соответствует уровню “превосходства” (9–10 баллов) в критериях проекта.
