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
