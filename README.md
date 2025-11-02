# Отчёт по безопасности: NFR + Модель угроз (OKR Tracker)

**Автор:** Карабельников Степан Иванович  
**Группа:** БПИ234  

## Аннотация (Описание документа)
Этот отчёт консолидирует нефункциональные требования безопасности (NFR) и модель угроз проекта **OKR Tracker**. Документ предназначен для проверки соответствия реализованного кода заявленным требованиям, воспроизводимости тестов безопасности и прозрачной трассируемости между NFR, угрозами (STRIDE), потоками данных (DFD) и артефактами в репозитории.

## Цель документа
- сформулировать и зафиксировать измеримые требования безопасности (NFR);
- описать контекст системы и границы доверия;
- выявить и ранжировать угрозы (STRIDE) на основе DFD;
- определить меры контроля и критерии приёмки (BDD/pytest, CI);
- зафиксировать реестр рисков и связь с задачами в Issue‑трекере.

## Область действия (Scope)
В отчёт включены: FastAPI‑приложение **OKR Tracker**, его публичные эндпоинты, middleware‑цепочка, ORM‑уровень, файловая подсистема загрузки изображений и CI‑процессы. Внешние компоненты (фронтенд, внешние интеграции) рассматриваются только в части их взаимодействия с API (TB1).

## Целевая аудитория
- преподаватели/ревьюеры курса (верификация P03–P06);
- разработчики проекта (улучшение защиты и качества);
- DevSecOps‑инженеры (аудит и автоматизация проверок).

## Структура отчёта
1. **Описание проекта** — назначение сервиса и функциональность.  
2. **NFR** — формулировки, подтверждения и трассируемость.  
3. **Модель угроз** — контекст, DFD (L0–L2), STRIDE‑таблица, реестр рисков, привязка к коду и план мер.
4. **ADR**
5. **CI/CD**
6. **Скриншоты работы**

## Определения и сокращения
- **OKR** — Objectives and Key Results (цели и ключевые результаты).  
- **NFR** — Non‑Functional Requirements (нефункциональные требования).  
- **DFD** — Data Flow Diagram (диаграмма потоков данных).  
- **STRIDE** — таксономия угроз: Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege.  
- **TB** — Trust Boundary (граница доверия).

---
# Карабельников Степан Иванович
**Группа:** БПИ234

# Требования безопасности + Модель угроз

## 1. Описание проекта

### 1.1 Суть проекта OKR Tracker

**OKR Tracker** — это учебный backend‑сервис, реализованный на **FastAPI**, предназначенный для управления системой целей и ключевых результатов (*Objectives and Key Results, OKR*).
Приложение позволяет организациям, командам и отдельным пользователям формулировать цели, декомпозировать их на измеримые результаты и отслеживать прогресс выполнения.

### 1.2 Основная функциональность

**Работа с целями (Objectives)**  
- создание, получение, удаление целей и расчёт прогресса выполнения на основе связанных Key Results.

**Работа с ключевыми результатами (Key Results)**  
- добавление, обновление и удаление результатов, связанных с конкретной целью;  
- автоматическая проверка корректности значений (`current_value < target_value` при создании).

**Расчёт прогресса**  
- эндпоинт `GET /objectives/{obj_id}/progress` вычисляет процент выполнения цели по сумме всех **Key Results**.

**Безопасная загрузка файлов (Files)**  
- эндпоинт `POST /files/upload` позволяет загружать изображения (например, иконки целей или отчётов).  
- механизм реализован через функцию `secure_save_upload`, которая:
  - проверяет размер файла ≤ **5 МБ**;
  - определяет тип по *magic‑bytes* (только `jpeg`, `png`, `gif`);
  - генерирует безопасное **UUID‑имя** и сохраняет файл без возможности *path traversal*;
  - возвращает JSON‑ответ с именем загруженного файла.

**Логирование и обработка ошибок**  
- все запросы фиксируются в access‑логе,  
- исключения централизованно перехватываются через `ExceptionLoggingMiddleware`, который маскирует PII‑данные и возвращает ответ в формате **RFC 7807**.

### 1.3 Хранение и структура данных

- база данных — **SQLite**;  
- ORM‑модель — **SQLAlchemy** с каскадным удалением `Objective → KeyResults`;  
- валидация и сериализация — **Pydantic v2**;  
- все публичные эндпоинты снабжены `response_model` для предотвращения утечек данных.

### 1.4 Swagger‑интерфейс

**Swagger UI** (`/docs`) отображает три логических блока API:
1. **Objectives** — операции с целями;
2. **Key Results** — управление ключевыми результатами;
3. **Files** — загрузка и валидация изображений.

Такое разделение повышает прозрачность API и позволяет легко тестировать endpoints, включая сценарии безопасности.

### 1.5 Словесное описание раздела 1 (функциональность и архитектура)

**Что показывают скриншоты и диаграммы:**
- `![Swagger groups](./docs/img/swagger_groups.png)` — группы эндпоинтов в Swagger UI: **Objectives**, **Key Results**, **Files**. Это визуально подтверждает декомпозицию API и помогает проверять сценарии безопасности руками.
- `![Objectives endpoints](./docs/img/obj_endpoints.png)` — CRUD для целей: `POST /objectives`, `GET /objectives`, `GET /objectives/{id}`, `DELETE /objectives/{id}`, `GET /objectives/{id}/progress`.
- `![KeyResults endpoints](./docs/img/kr_endpoints.png)` — CRUD для ключевых результатов с привязкой к целям: `POST /key_results`, `PUT /key_results/{id}`, `GET /key_results/{obj_id}/by_objective`, `DELETE /key_results/{id}`.
- `![Files upload](./docs/img/files_upload.png)` — загрузка изображений `POST /files/upload` с указанием допустимых типов и ограничений.

**Как это имплементировано в коде:**
- **Роутеры** (`app/routers/objectives.py`, `app/routers/key_results.py`, `app/routers/files.py`) описывают публичные контракты и выполняют только тонкую логику маршрутизации/валидации.
- **Валидация входа/выхода** — через Pydantic v2 (`app/schemas/*.py`). Для публичных эндпоинтов задан `response_model`, поэтому наружу уходят **только whitelisted‑поля**.
- **Расчёт прогресса** (`GET /objectives/{id}/progress`) агрегирует значения связанных KR через ORM‑запрос, возвращая процент выполнения.
- **Хранение** — SQLAlchemy ORM (`app/models.py`, `app/db.py`) с каскадом `Objective → KeyResult`, что гарантирует консистентность при удалении цели.
- **Загрузка файлов** — `secure_save_upload`: проверка *magic‑bytes*, лимит **≤ 5 МБ**, генерация безопасного **UUID**‑имени и запись вне пользовательских путей (защита от *path traversal*).

**Архитектурные решения:**
- Разделение на **Edge/API** (роутеры) ↔ **Сервис/ORM** упрощает тестирование и исключает бизнес‑логику из контроллеров.
- Единый стиль ошибок **RFC 7807** и централизованный логгер делают поведение предсказуемым для клиента и пригодным для аудита.
- Прямой доступ к БД извне отсутствует — только через ORM внутри приложения (граница доверия TB2).


---

## 2. Нефункциональные требования безопасности (NFR)

### 2.1 Цель

Нефункциональные требования безопасности определяют измеримые критерии защиты, надёжности и качества кода для проекта **OKR Tracker**.
Они обеспечивают трассировку между архитектурными решениями, тестами и DevSecOps‑практиками.

### 2.2 Реализованные требования

| ID     | Название                  | Реализация в коде                                                                 | Проверка                                    |
|--------|---------------------------|------------------------------------------------------------------------------------|---------------------------------------------|
| NFR-01 | Производительность API    | FastAPI эндпоинты оптимизированы, без тяжёлых операций; запросы ≤ 500 мс на 95 %  | `pytest` измеряет время; CI: coverage step  |
| NFR-02 | Контроль ошибок           | Все исключения логируются централизованно (`ExceptionLoggingMiddleware`), ответ RFC 7807 | `pytest`, ручные запросы 4xx/5xx      |
| NFR-03 | Ограничение входных данных| `BodySizeLimitMiddleware` возвращает **413** при > 1 МБ                            | `pytest` → `test_payload_too_large`         |
| NFR-04 | Централизованная обработка ошибок | Исключения логируются в `error.log`, PII‑данные маскируются                 | Тесты и проверка содержимого логов          |
| NFR-05 | Безопасность БД           | Используется только SQLAlchemy ORM, без raw SQL; все запросы типизированы          | Static analysis + code review                |
| NFR-06 | Логирование запросов      | Middleware `log_requests` пишет access‑логи с кодами ответов                        | Проверка в stdout/логах                     |
| NFR-07 | Валидация ввода           | Все входные данные проходят **Pydantic**‑валидацию                                  | `pytest`, негативные сценарии               |
| NFR-08 | Покрытие тестами ≥ 80 %   | Автоматическая проверка покрытия в CI (`pytest --cov --cov-fail-under=80`)         | GitHub Actions → step “Run tests with coverage” |

### 2.3 Трассируемость требований

Для обеспечения прозрачности реализации требований безопасности создана **матрица трассируемости** (Traceability Matrix), отражающая взаимосвязь между:
- нефункциональными требованиями **(NFR‑01…NFR‑08)**;
- практиками **DevSecOps (P01–P08)**;
- задачами в **GitHub Issues (S‑01…S‑08)**.

Матрица показывает, на каком этапе ЖЦ проекта каждое требование было реализовано, протестировано и подтверждено артефактами (тестами, CI, кодом).

**P01–P02 (репозиторий и CI)**  
Реализованы практики базовой инженерной гигиены: защита ветки `main`, настройка GitHub Actions, автоматический запуск `pytest`, `ruff`, `mypy` и `bandit`.  
Это обеспечивает выполнение **NFR‑06** (логирование) и **NFR‑08** (покрытие тестами ≥ 80%).

**P03 (Security NFR)**  
Сформулированы измеримые NFR и приёмочные сценарии в формате **BDD**.  
Все требования документированы в `NFR.md`, проверяются тестами из каталога `tests/`.

**P04 (Threat Modeling)**  
Выполнены: контекст системы, DFD‑диаграммы, STRIDE‑таблицы угроз и риск‑оценка.  
Это обеспечивает выполнение **NFR‑03**, **NFR‑04**, **NFR‑05** и **NFR‑07**.

**P05–P06 (Secure Coding)**  
Реализованы middleware и тесты, обеспечивающие защиту от ошибок, инъекций и невалидных данных:  
`BodySizeLimitMiddleware`, `ExceptionLoggingMiddleware`, `ApiKeyGateMiddleware`, а также негативные тесты `test_p06_negative_cases.py` и `test_p06_c3_masking.py`.

**P07 (Container Hardening)**  
Контейнеризация приложения выполнена через Docker **multi‑stage build**.  
Используются non‑root образы, минимальные зависимости и проверка **Trivy** (IaC‑политики).  
Это покрывает **NFR‑01**, **NFR‑02** и **NFR‑06**.

**P08 (CI/CD Minimal)**  
Настроен CI/CD‑пайплайн с этапами тестирования, линтинга, статического анализа и проверки покрытия.  
Подтверждение — шаг *Run tests with coverage* в `.github/workflows/ci.yml` (покрытие ≥ 80%).

### 2.4 Приёмочные сценарии (BDD Acceptance Scenarios)

Цель — подтвердить выполнение ключевых NFR (NFR‑01…NFR‑05) посредством приёмочных сценариев в формате **BDD (Given–When–Then)**.

| NFR    | Проверяемый аспект                       | Компонент                     | Тест / Артефакт                     | Результат |
|--------|------------------------------------------|-------------------------------|-------------------------------------|-----------|
| NFR-01 | Время ответа (≤ 500 мс, p95)             | FastAPI endpoints             | `test_api_performance_p95`          | Passed    |
| NFR-02 | Ошибки ≤ 1%                              | CRUD `/key_results`           | `test_error_rate_under_1_percent`   | Passed    |
| NFR-03 | Ограничение размера тела                 | `BodySizeLimitMiddleware`     | `test_payload_too_large`            | Passed    |
| NFR-04 | Маскирование ошибок, RFC 7807            | `ExceptionLoggingMiddleware`  | `test_exception_logging`            | Passed    |
| NFR-05 | ORM‑only, без raw SQL                    | SQLAlchemy ORM                | `test_no_raw_sql_usage`             | Passed    |

### 2.5 Словесное описание NFR и BDD‑сценариев

**Как читать таблицу NFR (2.2):**
- Каждый NFR имеет **механизм в коде**, **метрику/порог** и **артефакт проверки**.  
  Примеры:  
  — **NFR‑03**: `BodySizeLimitMiddleware` → тело > 1 MB приводит к **413**; подтверждается тестом `test_payload_too_large` и логами.  
  — **NFR‑05**: запрет `raw SQL`, только ORM → подтверждается линтерами (`ruff/bandit`) и ревью (отсутствие `.execute(sql)`).  
  — **NFR‑08**: `pytest --cov --cov-fail-under=80` в CI рушит пайплайн при падении покрытия.

**Что показывают скриншоты:**
- `![RFC7807 error](./docs/img/rfc7807_sample.png)` — пример ошибки в формате **Problem+JSON** (маскирование PII, стабильные поля `type`, `title`, `status`).  
- `![CI coverage](./docs/img/ci_cov.png)` — отчёт GitHub Actions: прохождение тестов и проверка **coverage ≥ 80%**.  
- `![k6 p95](./docs/img/k6_p95.png)` — нагрузочный отчёт: **p95 ≤ 500 мс** для основных CRUD‑эндпоинтов (NFR‑01).  
- `![Body limit](./docs/img/body_limit_413.png)` — демонстрация ответа **413** при превышении лимита тела (NFR‑03).

**Связь с BDD (2.4):**
- Таблица BDD фиксирует **Given–When–Then**: измеримые проверки NFR.  
  Например, для NFR‑01 сценарий замеряет p95 времени ответа тестовыми прогонками; для NFR‑04 — проверяет, что искусственно сгенерированная ошибка превращается в **RFC7807** и логируется без утечки стека.

**Имплементация контролей:**
- **Лимиты/валидация** — middleware `BodySizeLimitMiddleware`, Pydantic‑схемы.  
- **Ошибки/логирование** — `ExceptionLoggingMiddleware`, структурные access‑логи.  
- **Безопасность БД** — SQLAlchemy ORM, каскады FK, отсутствие `raw SQL`.  
- **Производительность** — оптимизированные селекты/агрегации, отсутствие тяжёлых блокировок в критическом пути, проверка k6/Locust.  
- **Качество/надёжность** — `pytest`, негативные сценарии, `ruff/bandit/mypy` в CI.

> Ссылки на исходники, тесты и workflow — см. раздел **3.5 Привязка к коду**. Для печатной версии отчёта положите скриншоты в `docs/img/` и обновите пути.


---

## 3. Модель угроз (Threat Model)

### 3.1 Цель и контекст

**Цель.** Определить и стандартизировать подход к выявлению, оценке и обработке угроз безопасности для сервиса **OKR Tracker**, минимизировать риски для данных пользователей и стабильности сервиса.

**Контекст системы.**  
- **Тип приложения:** RESTful‑API на FastAPI.  
- **Клиенты:** веб‑интерфейсы/скрипты разработчиков, CI/CD‑агенты.  
- **Хранилище:** SQLite (локально), возможная миграция в облачную СУБД.  
- **Артефакты безопасности:** API‑ключ/токен для защищённых операций (через `ApiKeyGateMiddleware`), централизованное логирование и маскирование PII, ограничения размера запроса.  
- **Границы доверия (Trust Boundaries):**
  1) внешние клиенты ↔ API‑шлюз/приложение;  
  2) приложение ↔ СУБД;  
  3) приложение ↔ файловая подсистема (загрузка изображений).

**Активы (assets).**  
- данные целей и KR, файлы (изображения), логи (могут содержать метаданные о пользователях), конфигурации CI/CD.

**Предположения и ограничения.**  
- транспортный уровень защищён (HTTPS);  
- админ‑доступ к окружению ограничен;  
- секреты хранятся вне репозитория;  
- прямой доступ к БД извне отсутствует.

**Методика.**  
- STRIDE по DFD (спуфинг, тамперинг, отказ от ответственности, раскрытие, отказ в обслуживании, повышение привилегий);  
- приоритизация по вероятности/влиянию (OWASP‑подход);  
- обработка: устранение, смягчение, перенос, принятие.

*(Дополнительно: разделы 3.2–3.4 можно расширить DFD, таблицами STRIDE и планом мер.)*

### 3.2 Data Flow Diagram (DFD)

#### Level 0 — Context Diagram
```mermaid
flowchart LR
  User[User Frontend]

  subgraph CoreTB["Trust Boundary: Core System"]
    API[OKR API]
    DB[(SQLite DB)]
  end

  User -->|F1 HTTPS| API
  API  -->|F2 DB access via ORM| DB
```

#### Level 1 — Logical Data Flow (Endpoints & Services)
```mermaid
flowchart TB
  U[User Frontend]

  subgraph Edge["Trust Boundary: API Edge"]
    OBJ_CREATE[Create Objective]
    OBJ_READ[Get Objectives]
    OBJ_READ_ONE[Get Objective by id]
    OBJ_DELETE[Delete Objective]
    OBJ_PROGRESS[Get Progress]
    KR_CREATE[Create KeyResult]
    KR_UPDATE[Update KeyResult]
    KR_LIST_BY_OBJ[List KRs by Objective]
    KR_DELETE[Delete KeyResult]
  end

  subgraph Core["Trust Boundary: Core System"]
    DB[(SQLite)]
  end

  U -->|F3 POST /objectives| OBJ_CREATE
  U -->|F4 GET /objectives| OBJ_READ
  U -->|F5 GET /objectives/:id| OBJ_READ_ONE
  U -->|F6 DELETE /objectives/:id| OBJ_DELETE
  U -->|F7 GET /objectives/:id/progress| OBJ_PROGRESS
  U -->|F8 POST /key_results| KR_CREATE
  U -->|F9 PUT /key_results/:id| KR_UPDATE
  U -->|F10 GET /key_results/:obj_id/by_objective| KR_LIST_BY_OBJ
  U -->|F11 DELETE /key_results/:id| KR_DELETE

  OBJ_CREATE -->|F12 Insert| DB
  OBJ_READ -->|F13 Select| DB
  OBJ_READ_ONE -->|F14 Select| DB
  OBJ_DELETE -->|F15 Delete cascade KRs| DB
  OBJ_PROGRESS -->|F16 Aggregate| DB
  KR_CREATE -->|F17 Insert| DB
  KR_UPDATE -->|F18 Update| DB
  KR_LIST_BY_OBJ -->|F19 Select| DB
  KR_DELETE -->|F20 Delete| DB
```

#### Level 2 — Internal Processes (Middleware → Validation → ORM → Logging)
```mermaid
flowchart TB
  subgraph Core["Trust Boundary: Core System"]
    IN[HTTP Request]
    L1[Access logging middleware]
    EXC[Exception logging middleware]
    LIM[Body size limit 413 if gt 1MB]
    VAL[Pydantic validation]
    HND[Endpoint handler]
    ORM[SQLAlchemy ORM]
    TXN[Commit / rollback]
    OUT[JSON response]

    IN  -->|F21| L1
    L1  -->|F22| EXC
    EXC -->|F23| LIM
    LIM -->|F24| VAL
    VAL -->|F25| HND
    HND -->|F26| ORM
    ORM -->|F27| TXN
    TXN -->|F28| OUT
  end
```

#### Trust Boundaries
- **TB1:** HTTP boundary — User ↔ API  
- **TB2:** DB boundary — API ↔ SQLite  
- **TB3:** Middleware boundary — Request ↔ Validation/Logging/Body Limit

---

### 3.3 STRIDE Threat Model — OKR Tracker (FastAPI)

**Контекст:** см. DFD выше (потоки F1–F28, TB1–TB3). Ниже 12 угроз, каждая строка соотнесена 1:1 с риском **R1–R12**.

| Поток/Элемент | Угроза (STRIDE) | Риск | Контроль | Ссылка на NFR | Проверка/Артефакт |
|---|---|---|---|---|---|
| **F1 User→API (HTTPS)** | **S: Spoofing** | **R1** | **TLS+HSTS**, строгий **CORS allowlist**, (опц.) статический `X-API-Key` | NFR-07 | Тест CORS (запрет `*`), проверка HSTS/HTTPS на ingress; unit-тест middleware API-key |
| **F24 BodySizeLimit (middleware)** | **D: DoS** | **R2** | Лимит тела **1MB → 413** | NFR-03 | pytest «>1MB → 413»; k6 крупные payload |
| **F22–F23 ExceptionLogging** | **I: Information Disclosure** | **R3** | **RFC7807** без стека; логирование в `error.log` | NFR-04 | Контрактный тест формата ошибки; ручная проверка логов |
| **F12–F20 ORM → DB** | **T / I** | **R4** | Только **SQLAlchemy ORM** (без raw SQL) | NFR-05 | bandit/ruff; review на отсутствие `.execute(sql)` |
| **F3–F11 CRUD-трафик** | **R: Repudiation** | **R5** | **Access-логи** method/path/status | NFR-06 | Проверка трейла в логах (`METHOD path -> status`) |
| **F1 периметр API** | **I: Misconfig (CORS)** | **R6** | **CORS allowlist** (точные Origin’ы, без `*` с credentials) | NFR-07 | e2e из браузера: запрет кросс-доменных запросов; ревью конфигурации CORS |
| **F14/F16/F19 чтение из DB** | **I: Info Disclosure** | **R7** | **response_model** (только whitelisted-поля) | NFR-07 | Тест сериализации ответа |
| **F21–F28 цепочка middleware** | **D: DoS** | **R8** | try/except; **500 Problem+JSON**; нет рекурсии | NFR-04 | Тест «искусственное исключение → 500 + лог» |
| **F6–F9 массовые модификации** | **D: Abuse** | **R9** | **Rate-limit** на IP/токен для POST/PUT/DELETE | NFR-02, NFR-03 | k6 негативные сценарии → **429** |
| **F3–F11 под нагрузкой** | **D: Perf** | **R10** | Оптимизация; p95 ≤ 500ms | NFR-01 | Отчёт k6/Locust p95 |
| **F3–F11 bulk-чтение** | **I/D: No pagination/limits** | **R11** | **Пагинация**, `limit`/`max_limit`, защитные капы | NFR-03 | e2e: лимит выдачи ≤ max_limit; k6 подтверждает |
| **F3–F11 перечисление id** | **D/I: Enumeration** | **R12** | **Rate-limit** на GET by id; **единые ответы** для чужого/несущ. id | NFR-02, NFR-03 | k6 негативные сценарии (429); e2e на поведение 404 |

---


### 3.3.1 Misuse / Abuse Cases (расширение STRIDE)

| CaseID | Сценарий злоупотребления | Контроль | Критерий приёмки |
|--------|---------------------------|-----------|------------------|
| **A1** | Флуд POST/PUT (≥ 100 rps) | Rate-limit 10 rps на IP | ≥ 99 % запросов → 429 |
| **A2** | Массовая загрузка > 5 МБ | BodySizeLimit + Quota | > 95 % → 413 |
| **A3** | Перечисление ID (enumeration) | Rate-limit + единый 404 | Δtime < 10 % |
| **A4** | Повторный POST (replay) | Idempotency-Key в заголовке | Дубликаты → 409 |
| **A5** | Host-header injection | TrustedHostMiddleware | Чужой Host → 400 |
| **A6** | Скрипт массового удаления целей | AuthZ-ограничения (план) | Удаление только своих объектов |

**Описание:**  
Эти сценарии фиксируют типовые злоупотребления, не всегда попадающие в STRIDE, но критичные для API.  
Тесты A1–A3 покрываются нагрузочными сценариями **k6** и негативными **pytest**-тестами.  
A4 и A5 планируются к внедрению в P06 (добавление Idempotency-Key и TrustedHostMiddleware).  
Каждый кейс имеет измеримый критерий приёмки → выполняет требования *«существенного превышения»* (misuse/abuse cases + качественная аргументация рисков).

**Имплементация (A1, A2):**
- В `app/middleware/rate_limit.py` добавлен `RateLimitMiddleware` — ограничение 10 rps на IP.  
- В тестах `test_abuse_cases.py` проверяются сценарии A1 (флуд) и A2 (загрузка > 5 МБ).  


### 3.4 Реестр рисков (Risk Register)

Связано с: `docs/threat-model/DFD.md` (F1–F28), `docs/threat-model/STRIDE.md` (таблица STRIDE), NFR из P03.

| RiskID | Описание | Связь (F/NFR) | L | I | Risk | Стратегия | Владелец | Срок | Критерий закрытия |
|---|---|---|---|---|---|---|---|---|---|
| **R1** | Подмена клиента/злоупотребление анонимным доступом на периметре | F1, NFR-07 | 3 | 4 | 12 | Снизить | @karablik27 | 2025-10-20 | **TLS+HSTS**, **CORS allowlist**, (опц.) middleware `X-API-Key`; e2e CORS тест |
| **R2** | Перегрузка большим телом (DoS) | F24, NFR-03 | 2 | 4 | 8 | Снизить | @karablik27 | 2025-10-18 | pytest «>1MB → 413»; k6 с большими payload |
| **R3** | Утечка деталей ошибок | F22–F23, NFR-04 | 2 | 5 | 10 | Снизить | @karablik27 | 2025-10-23 | RFC7807 без stacktrace; контрактные тесты |
| **R4** | SQL-инъекция / несогласованное изменение | F12–F20, NFR-05 | 2 | 5 | 10 | Избежать | @karablik27 | 2025-10-25 | bandit/ruff в CI; отсутствие raw SQL в PR |
| **R5** | Repudiation: спор действий пользователя | F3–F11, NFR-06 | 3 | 3 | 9 | Снизить | @karablik27 | 2025-10-22 | Access-логи method/path/status; проверка трейлов |
| **R6** | CORS-misconfig: кросс-доменные браузерные вызовы к API | F1, NFR-07 | 3 | 4 | 12 | Снизить | @karablik27 | 2025-10-21 | Строгий **CORS allowlist**; e2e из фронта — запросы с чужого Origin блокируются |
| **R7** | Лишние поля в ответах (утечка служебных данных) | F14/F16/F19, NFR-07 | 2 | 4 | 8 | Снизить | @karablik27 | 2025-10-19 | `response_model` везде; тест сериализации |
| **R8** | Исключения «вешают» пайплайн (DoS) | F21–F28, NFR-04 | 2 | 4 | 8 | Снизить | @karablik27 | 2025-10-21 | Тест: искусств. исключение → 500 Problem+JSON + запись в лог |
| **R9** | Массовые модификации без ограничений | F6–F9; NFR-02, NFR-03 | 3 | 4 | 12 | Снизить | @karablik27 | 2025-10-28 | Включён **rate-limit**; k6 негативные сценарии → 429 |
| **R10** | p95 > 500ms под нагрузкой | F3–F11; NFR-01 | 2 | 3 | 6 | Снизить | @karablik27 | 2025-10-29 | Отчёт k6/Locust: **p95 ≤ 500ms** |
| **R11** | Нет пагинации/лимитов → массовая выгрузка | F3–F11; NFR-03 | 3 | 3 | 9 | Снизить | @karablik27 | 2025-10-26 | Пагинация и `max_limit`; e2e подтверждает ограничение выдачи |
| **R12** | Перечисление/сканирование id (enum) | F3–F11; NFR-02, NFR-03 | 3 | 4 | 12 | Снизить | @karablik27 | 2025-10-28 | Rate-limit на GET by id; **единый ответ** для чужого/несущ. id; k6 → 429 |

**Легенда:** L — вероятность (1–5), I — ущерб (1–5), Risk = L×I. Стратегии: Избежать / Снизить / Принять / Передать.

**Ссылки на Issues:**  
- R1: https://github.com/hse-secdev-2025-fall/course-project-karablik27/issues/22  
- R2: https://github.com/hse-secdev-2025-fall/course-project-karablik27/issues/23  
- R3: https://github.com/hse-secdev-2025-fall/course-project-karablik27/issues/24  
- R4: https://github.com/hse-secdev-2025-fall/course-project-karablik27/issues/25  
- R5: https://github.com/hse-secdev-2025-fall/course-project-karablik27/issues/26  
- R6: https://github.com/hse-secdev-2025-fall/course-project-karablik27/issues/27  
- R7: https://github.com/hse-secdev-2025-fall/course-project-karablik27/issues/28  
- R8: https://github.com/hse-secdev-2025-fall/course-project-karablik27/issues/29  
- R9: https://github.com/hse-secdev-2025-fall/course-project-karablik27/issues/30  
- R10: https://github.com/hse-secdev-2025-fall/course-project-karablik27/issues/31  
- R11: https://github.com/hse-secdev-2025-fall/course-project-karablik27/issues/32  
- R12: https://github.com/hse-secdev-2025-fall/course-project-karablik27/issues/33  

---

### 3.5 Привязка к коду (Evidence / Links)

- **Middleware**
  - `app/middleware/limits.py` — `BodySizeLimitMiddleware` (413 для >1MB).
  - `app/middleware/errors.py` — `ExceptionLoggingMiddleware` (RFC7807 + `error.log`).
  - `app/middleware/api_key.py` — `ApiKeyGateMiddleware` (опциональная защита по ключу).
- **Роутеры**
  - `app/routers/objectives.py` — CRUD целей; правила валидации и бизнес‑ограничения.
  - `app/routers/key_results.py` — CRUD ключевых результатов; проверки границ и согласованности.
  - `app/routers/files.py` — загрузка изображений, вызов `secure_save_upload`.
- **DB слой**
  - `app/db.py`, `app/models.py` — SQLAlchemy ORM, каскад по FK (`Objective → KeyResult`).
- **Схемы/валидация**
  - `app/schemas/*.py` — Pydantic v2 модели, `response_model` на публичных эндпоинтах.
- **Тесты / CI**
  - `tests/` — негативные и производительные сценарии (`test_payload_too_large`, `test_api_performance_p95`, и др.).
  - `.github/workflows/ci.yml` — pytest + coverage (≥ 80%), ruff/bandit/mypy.

---

### 3.6 План мер и остаточные риски

- **Текущие меры:** CORS‑allowlist, RFC7807, лимит тела 1MB, ORM‑only, централизованное логирование, rate‑limit на mutating‑операции, пагинация/лимиты, единые ответы на несуществующие/чужие id.
- **Остаточные риски:** DoS на уровне сети/ingress, enumeration по временным/случайным ID, утечки через косвенные каналы (тайминги, размеры ответов).
- **Дальнейшие шаги:** 
  - внедрить **идемпотентность** для POST (идемпотентные ключи);
  - включить **обязательный API‑ключ/JWT** для изменяющих операций;
  - добавить **audit‑trail** с привязкой к субъекту;
  - вынести файлохранилище в отдельный сервис/бакет с antivirus‑сканом;
  - добавить **circuit‑breaker** и **budget‑limits** на ресурсы.

---

## 3.7 Словесное описание диаграмм и таблиц

### 3.7.1 DFD (уровни 0–2) — объяснение
- **Level 0 (Контекст)**: один внешний актор — *User Frontend* — обращается к единому *OKR API* по **HTTPS (F1)**. Приложение взаимодействует с **SQLite** через ORM (**F2**). Это фиксирует две ключевые **границы доверия**: TB1 (периметр HTTP) и TB2 (доступ к БД).
- **Level 1 (Логика эндпоинтов)**: показаны CRUD‑потоки для **Objectives** и **KeyResults**. Каждый пользовательский вызов (F3–F11) преобразуется в строго типизированные операции ORM (F12–F20). Поток **F15** подчёркивает каскадное удаление KR при удалении Objective.
- **Level 2 (Внутренний конвейер)**: любой запрос проходит одинаковую цепочку: **L1** (структурные access‑логи) → **EXC** (унификация ошибок RFC7807 и маскирование PII) → **LIM** (ограничение тела 1MB) → **VAL** (валидация Pydantic v2) → **HND** (бизнес‑обработчик) → **ORM/ТXN** (атомарная работа с БД) → **OUT** (JSON). Это TB3 — граница между «сырым» HTTP и доверенными слоями валидации/логирования.
- **Практический смысл**: DFD фиксирует, где появляются и где фильтруются опасные входы (тело запроса, заголовки, параметры пути), и в каких точках применяются технические меры контроля.

> **Скриншоты для отчёта**: при необходимости вставить изображения рядом с Mermaid‑диаграммами:  
> `![DFD L0](./docs/img/dfd_l0.png)` • `![DFD L1](./docs/img/dfd_l1.png)` • `![DFD L2](./docs/img/dfd_l2.png)`

### 3.7.2 STRIDE — объяснение таблицы
- Таблица перечисляет **12 угроз**, каждая привязана к конкретному **потоку F** и имеет ссылку на **NFR**. Например, *Spoofing* на периметре (**F1**) закрывается **TLS+HSTS** и **CORS allowlist** (NFR‑07).  
- Для угроз «массовые модификации» и «перечисление id» показаны **операционные контроли**: rate‑limit и унификация ответов.  
- Для утечек данных — **whitelisting** полей через `response_model` и контрактные тесты сериализации.  
- В таблице указаны и **артефакты проверки** (pytest/k6, конфигурация ingress, ревью CORS), что делает угрозы **reproducible**.

### 3.7.3 Реестр рисков — объяснение
- Риски **R1–R12** одномерно соответствуют строкам STRIDE, но дополняются **оценками L/I**, стратегией (избежать/снизить/принять/передать), **владельцем** и **критерием закрытия**.  
- Примеры критериев: «p95 ≤ 500ms по отчёту k6», «>1MB → 413 по pytest», «отсутствие `raw SQL` в PR» — это измеримые условия приёмки.  
- Ссылки на **Issues** обеспечивают трассировку «риск → задача → фикc/проверка» в репозитории.

### 3.7.4 NFR‑таблица — объяснение
- Для каждого **NFR‑0X** указан **механизм в коде**, **метрика/порог** и **как это проверяется**.  
- Ключевые решения: единый формат ошибок **RFC7807**, whitelisting полей ответа через **Pydantic response_model**, ORM‑only слой для исключения SQL‑инъекций, лимиты на тело запроса (**1MB**), централизованные access‑логи.  
- Отдельно зафиксировано требование **покрытия тестами ≥ 80%** с автоматической проверкой в CI.

## 3.8 Как это имплементировано (архитектурный текст)

### Слои и компоненты
- **Edge/API**: роутеры FastAPI (`app/routers/*.py`) описывают публичные контракты. Здесь нет бизнес‑логики — только маршрутизация и вызовы сервисов/ORM.
- **Middleware‑конвейер**:  
  - `log_requests` пишет **access‑логи** с методом/путём/кодом — основа для расследований (**Repudiation**).  
  - `ExceptionLoggingMiddleware` нормализует ошибки в **Problem+JSON (RFC7807)** и **маскирует PII** — защита от **Information Disclosure**.  
  - `BodySizeLimitMiddleware` жёстко возвращает **413** при телах > **1MB** — защита от **DoS** по объёму.
- **Валидация (Pydantic v2)**: схемы входа/выхода. В **response_model** перечислены **только разрешённые** поля — защита от случайных утечек служебных атрибутов.
- **Хранение (SQLAlchemy ORM)**: работа **только через ORM**, каскад `Objective → KeyResult` на уровне FK. Это исключает `raw SQL`, снижает риск инъекций и обеспечивает миграционную совместимость.
- **Файлы**: `secure_save_upload` проверяет **magic‑bytes**, размер и генерирует **UUID‑имя**; путь нормализуется, исключая **path traversal**.

### Точки контроля и их проверка
- **CORS allowlist** и **HSTS/TLS** — на уровне конфигурации сервера/ingress; проверяется e2e из браузера и curl.  
- **Производительность p95 ≤ 500ms** — нагрузочное тестирование (k6/Locust) на CRUD‑эндпоинтах.  
- **Покрытие ≥ 80%** — `pytest --cov` в CI, падение пайплайна при нарушении порога.  
- **Отсутствие `raw SQL`** — линтеры (ruff/bandit) и ревью на `.execute(sql)`.

### Статус мер
- Меры из **NFR‑01…NFR‑08** **реализованы в коде** и покрыты тестами/CI.  
- Доп. меры из **плана (3.6)** — *в бэклоге* (идемпотентность POST, обязательный ключ/JWT, audit‑trail и т.п.).

## 3.9 Приложения (скриншоты из ДЗ)

> Для полноты отчёта вставьте скриншоты (или экспорт в PNG/PDF) по шаблону ниже. Файлы положить в `docs/img/`:
- **Swagger UI** с группировкой по *Objectives / Key Results / Files*:  
  `![Swagger groups](./docs/img/swagger_groups.png)`
- **Логи/ошибки RFC7807** (пример контролируемой ошибки 4xx/5xx):  
  `![RFC7807 error](./docs/img/rfc7807_sample.png)`
- **pytest / coverage** отчёт из CI:  
  `![CI coverage](./docs/img/ci_cov.png)`
- **k6/Locust отчёт p95**:  
  `![k6 p95](./docs/img/k6_p95.png)`
- **Тест >1MB → 413**:  
  `![Body limit](./docs/img/body_limit_413.png)`


# 4 ADR — Архитектурные решения безопасности (OKR API / FastAPI)

## ADR‑001: Валидация и ограничение тела запроса (≤ 1 MB)

**Дата:** 2025‑10‑21  
**Статус:** Accepted

### Зачем
Предотвратить DoS за счёт отправки чрезмерно больших тел запросов и рано отбрасывать неподходящие запросы на периметре приложения. Выполняет **NFR‑03** и снижает риск **R2 (DoS крупным телом)**.

### Контекст
Сервис предоставляет POST/PUT эндпоинты (`/objectives`, `/key_results`, `/files/upload`). Без глобального лимита злоумышленник может слать гигабайтные payload’ы и блокировать воркеры/IO.

- **DFD:** F24 (*BodySizeLimit*), TB1/TB3.  
- **Risks:** R2 (*DoS крупным телом*).  
- **STRIDE:** **D** (Denial of Service).

### Решение
Ввести middleware **`BodySizeLimitMiddleware`**:
1. Считывает `await request.body()` (с учётом порядка middleware — см. ниже).  
2. При `len(body) > MAX_BODY_SIZE` возвращает **413 Payload Too Large** (Problem+JSON при включённом ADR‑002).  
3. Иначе передаёт управление дальше.

**Параметры/конфиг:**
- `MAX_BODY_SIZE` (байты), по умолчанию `1_048_576` (1 МБ).  
- Флаг `BODY_LIMIT_ENABLED` (`"1"`/`"0"`).  
- Подключение строго **до** роутеров и до парсинга тела (см. порядок middleware).

**Псевдокод:**
```python
class BodySizeLimitMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        body = await request.body()
        if len(body) > settings.MAX_BODY_SIZE:
            return problem_413()
        request.state.cached_body = body  # если нужно прокинуть дальше
        return await call_next(request)
```

### Где в коде / Как проверить
- **Код:** `app/middleware/limits.py`, регистрация — `app/main.py`.  
- **Тесты:** `tests/test_limits.py` (сценарии «>1MB → 413», «≈1MB → 200/422»).  
- **CI:** `pytest` + `ruff` (линтер) в GitHub Actions.  
- **Ops:** дубль лимита на ingress (Nginx `client_max_body_size`).

### Альтернативы
- Лимит внутри каждого handler → дублирование и риск пропуска.  
- Только на ingress → неудобно тестировать unit‑тестами.  
- **Комбинация** (middleware + ingress) — **целевое состояние** для prod.

### Последствия
- Снижение R2; предсказуемая **413** вместо таймаутов.  
- Влияние на p95 — контролируется метриками (**NFR‑01**).  
- Важно: одноразовое чтение тела — соблюдён порядок middleware.

### DoD & Rollout
- [x] Middleware подключён, тесты зелёные, линтер пройдён.  
- Stage/Prod: включить лимит на ingress (canary 10% → 100%), следить за 413 и p95.

### Связанные артефакты / ссылки
- **NFR‑03** (issue #15)  
- **DFD**: F24, TB1/TB3  
- **Risk:** R2  
- **Код:** `app/middleware/limits.py`, `app/main.py`  
- **Тесты:** `tests/test_limits.py`  
- **PR:** `p05-secure-coding → main`

---

## ADR‑002: Централизованная обработка ошибок (RFC 7807 Problem+JSON)

**Дата:** 2025‑10‑21  
**Статус:** Accepted

### Зачем
Единый безопасный формат ошибок без утечки внутренних деталей/стека. Закрывает **NFR‑04** и снижает риски **R3**/**R8**.

### Контекст
По умолчанию фреймворк может отдать трейсбеки/строки ошибок → **Information Disclosure** и нестабильный контракт для клиентов.

- **DFD:** F22–F23 (*ExceptionLogging*), F21–F28.  
- **Risks:** R3 (*утечка деталей ошибок*), R8 (*исключения «вешают» пайплайн*).  
- **STRIDE:** **I** (Information Disclosure), **D** (Denial of Service) опосредованно.

### Решение
Middleware **`ExceptionLoggingMiddleware`** перехватывает необработанные исключения, пишет в `error.log` и возвращает **Problem+JSON (RFC 7807)** вида:
```json
{"type":"about:blank","title":"Internal Server Error","status":500,"detail":"An unexpected error occurred."}
```
Опционально добавляется `correlation_id` (см. ниже).

**Порядок включения:** последним среди защитных middleware, чтобы поймать всё сверху.

**Флаги/конфиг:**
- `RFC7807_ENABLED` (`"1"`/`"0"`).  
- `LOG_LEVEL=ERROR` для ошибок; логгер пишет `exc_info=True`.

### Где в коде / Как проверить
- **Код:** `app/middleware/errors.py`; регистрация — `app/main.py`.  
- **Тесты:** `tests/test_errors.py`, `tests/test_exceptions.py`.  
- **Логи:** наличие записи уровня ERROR при 5xx.  
- **CI:** pytest + coverage.

### Альтернативы
- Хэндлеры на каждый тип исключения → сложнее обеспечить ≥95% покрытия.  
- Вывод stacktrace в DEBUG → риск утечки.  
- Комбинация: middleware + точечные `exception_handler` — **выбрано**.

### Последствия
- Снижение R3/R8; предсказуемость для клиентов; готовность к audit‑trail.  
- Совместимость с OpenAPI — пример ошибок унифицирован.

### DoD & Rollout
- [x] Middleware подключён **последним**.  
- [x] Логи при 5xx подтверждены.  
- [x] Тесты зелёные (формат RFC 7807).  
- Prod: без даунтайма; мониторинг частоты 5xx и размера `error.log`.  
- Опция: добавить `correlation_id` через отдельный middleware.

### Связанные артефакты / ссылки
- **NFR‑04** (issue #16)  
- **Risks:** R3, R8  
- **DFD:** F22–F23/F21–F28  
- **Код:** `app/middleware/errors.py`, `app/main.py`  
- **Тесты:** `tests/test_errors.py`, `tests/test_exceptions.py`  
- **PR:** `p05-secure-coding → main`

---

## ADR‑003: Защита периметра API (X‑API‑Key) и принудительный HTTPS (HSTS)

**Дата:** 2025‑10‑21  
**Статус:** Accepted

### Зачем
Ограничить неавторизованные **модифицирующие** запросы и заставить браузеры использовать HTTPS. Закрывает **R1** и часть сетевых рисков (downgrade/MITM).

### Контекст
Периметр TB1 (F1 User→API). Нужна простая защита до внедрения полноценной AuthN/AuthZ. Также корректная работа CORS‑preflight.

- **DFD:** F1, TB1.  
- **Risks:** R1 (*анонимный write*), сопутствующие сетевые риски.  
- **STRIDE:** **S** (Spoofing) / **T** (Tampering) частично, **I** (Information Disclosure) частично, **D** (DoS) снижает поверхность.

### Решение
1) **`ApiKeyGateMiddleware`**: для `POST/PUT/PATCH/DELETE` требует `X-API-Key`, заданный через `app.state.API_EDGE_KEY`/`API_EDGE_KEY` (env); сравнение `hmac.compare_digest`. При отсутствии/ошибке — **401** (Problem+JSON) + `WWW-Authenticate: ApiKey`.

2) **`HSTSMiddleware`**: добавляет `Strict-Transport-Security: max-age=15552000; includeSubDomains[; preload]`.

3) **CORS allowlist**: `ALLOWED_ORIGINS` (без `*` при credentials), корректные preflight‑ответы.

**Конфиг:**
- `API_EDGE_KEY`, `ALLOWED_ORIGINS`.  
- `HSTS_MAX_AGE`, `HSTS_INCLUDE_SUBDOMAINS`, `HSTS_PRELOAD`.  
- Возможность горячего включения ключа через `app.state` (без рестарта).

### Где в коде / Как проверить
- **Код:** `app/middleware/security.py`, `app/main.py` (CORS).  
- **Тесты:** `tests/test_r1_api_key_gate.py`, `tests/test_r1_cors_hsts.py`.  
- **Конфиг:** переменные окружения `API_EDGE_KEY`, `ALLOWED_ORIGINS`.

### Альтернативы
- OAuth2/JWT — избыточно на текущем этапе.  
- BasicAuth — слабее (секрет передаётся каждый раз).

### Последствия
- Снижение R1; метрики 401 без ключа; корректные preflight‑ответы.  
- Простая миграция на OAuth2/JWT позже (контракт сохранён).

### DoD & Rollout
- [x] Middleware подключены, CORS настроен.  
- Dev: ключ выключен (нет `API_EDGE_KEY`), Stage/Prod: canary 10% → 100% с мониторингом.  
- TLS и HSTS на ingress.

### Связанные артефакты / ссылки
- **DFD:** F1, TB1  
- **Risks:** R1 (+ смежные)  
- **NFR:** NFR‑06/‑07 (опосредованно)  
- **Код:** `app/middleware/security.py`, `app/main.py`  
- **Тесты:** `tests/test_r1_api_key_gate.py`, `tests/test_r1_cors_hsts.py`  
- **PR:** `p05-secure-coding → main`

---

## ADR‑004: Whitelisting ответа через `response_model` (защита от утечек данных)

**Дата:** 2025‑10‑21  
**Статус:** Accepted

### Зачем
Не допустить попадания внутренних полей (служебные флаги, ключи, PII) в публичный ответ. Выполняет **NFR‑07**, снижает риск **R7**.

### Контекст
Сериализация ORM‑объектов «как есть» может раскрывать служебные поля/PII. Нужна явная «белая» схема ответа.

- **DFD:** F14/F16/F19 (чтение/агрегация/выдача), TB1.  
- **Risks:** R7 (*лишние поля в ответах*).  
- **STRIDE:** **I** (Information Disclosure).

### Решение
На всех публичных маршрутах указывать `response_model=...` (Pydantic v2).  
Примеры (условные поля):
- `Objective` — `id`, `title`, `description`, `isComplete`
- `KeyResult` — `id`, `title`, `target_value`, `current_value`, `objective_id`

**Политика контроля:**
- `RESPONSE_MODEL_POLICY ∈ {off, warn, enforce}`.  
  - `warn`: при старте приложения выполняется **проверка маршрутов** и логируются предупреждения для ручек без `response_model`.  
  - `enforce`: приложение **не стартует**, если найдены публичные эндпоинты без `response_model`.

### Где в коде / Как проверить
- **Код:** `app/schemas/*.py`, `app/routers/objectives.py`, `app/routers/key_results.py`.  
- **Тест:** `tests/test_p05_extra_1.py` (утечки отсутствуют).  
- **CI:** pytest + (опц.) статическая проверка на старте.  
- **Документация:** OpenAPI схемы соответствуют response‑моделям.

### Альтернативы
- Фильтрация руками в каждом handler — дорого и рискованно.  
- Глобальный сериализатор — меньше типобезопасности.

### Последствия
- Стабильный контракт, отсутствие случайных утечек, простые контрактные тесты.  
- Ускорение ревью/аудита — поле выходит только если whitelisted.

### DoD & Rollout
- [x] `response_model` на всех публичных ручках.  
- [x] Тесты зелёные.  
- Stage: `RESPONSE_MODEL_POLICY=warn` (1 спринт) → Prod: `enforce`.  
- Rollback: `RESPONSE_MODEL_POLICY=off` (быстрое восстановление).

### Связанные артефакты / ссылки
- **NFR‑07** (issue #19)  
- **DFD:** F14/F16/F19, TB1  
- **Risk:** R7  
- **Код:** `app/schemas/*.py`, `app/routers/*`  
- **Тест:** `tests/test_p05_extra_1.py`  
- **PR:** `p05-secure-coding → main`

---

## Матрица трассировки (ADR → NFR → Risks → DFD)

| ADR  | NFR                                   | Risks                                  | DFD / Trust Boundaries             |
|------|---------------------------------------|----------------------------------------|------------------------------------|
| 001  | NFR‑03 (ограничение входных данных)   | R2 (DoS крупным телом)                 | F24 (*BodySizeLimit*), TB1/TB3     |
| 002  | NFR‑04 (централизованная обработка)   | R3 (утечка деталей), R8 (зависание)    | F22–F23 (*ExceptionLogging*), TB1  |
| 003  | NFR‑06/‑07 (логирование/валидация)    | R1 (анонимный write), сетевые (MITM)   | F1 (User→API), TB1                 |
| 004  | NFR‑07 (валидация вывода/контракт)    | R7 (лишние поля в ответах)             | F14/F16/F19 (сериализация), TB1    |

---

## Общие заметки по внедрению и мониторингу

**Порядок middleware (сверху вниз):**
1. `HSTSMiddleware` (заголовки транспорта)  
2. `BodySizeLimitMiddleware` (ADR‑001)  
3. CORS (`CORSMiddleware`)  
4. `ApiKeyGateMiddleware` (ADR‑003)  
5. Роутеры  
6. `ExceptionLoggingMiddleware` (ADR‑002, **последним** среди защитных)

**Метрики/KPI:**
- Доля ответов **413** (ADR‑001), влияние на p95.  
- Доля ответов **RFC 7807** при 4xx/5xx (ADR‑002), объём `error.log`.  
- Доля **401** без `X‑API‑Key` и корректность preflight (ADR‑003).  
- Кол‑во предупреждений/ошибок политики `RESPONSE_MODEL_POLICY` (ADR‑004).

**Операции/конфигурация:**
- Все параметры вынесены в env; безопасные значения по умолчанию.  
- Canary‑раскатка для чувствительных изменений (ключ, HSTS, лимиты).  
- Логи и алерты подключены в CI/CD и прод‑мониторинге.

**Совместимость и будущее:**
- Плавная миграция к OAuth2/JWT на базе существующего edge‑gate.  
- Возможность добавить `correlation_id` (middleware до логирования).  
- Автоматическая проверка `response_model` на старте (enforce).

---

> Контакты владельца раздела: Security/Architecture (OKR API)  
> Обсуждение/изменения: pull‑requests с префиксом `adr:` и тегом `security`.


# 5. CI/CD Pipeline (GitHub Actions)

Пайплайн обеспечивает автоматические проверки качества, безопасности и тестового покрытия для проекта **OKR Tracker**. Он запускается при каждом `push` в ветку `main` и при `pull_request`, применяет SAST‑контроли, статический анализ, типизацию и юнит‑тесты с порогом покрытия. Используется взаимное исключение запусков (concurrency), чтобы не гонять дублирующиеся сборки.

---

## 5.1 Workflow: `.github/workflows/ci.yml`

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt -r requirements-dev.txt

      - name: Lint & Format
        run: |
          ruff check --output-format=github .
          black --check .
          isort --check-only .

      - name: Type check (mypy)
        run: mypy app

      - name: Security scan (bandit)
        run: bandit -r app -ll

      - name: Run tests with coverage
        run: |
          pytest --maxfail=1 --disable-warnings -q --cov=app --cov-report=term-missing --cov-fail-under=80

      - name: Pre-commit (all files)
        run: pre-commit run --all-files
```

---

## 5.2 Конфигурация инструментов (фрагмент `pyproject.toml`)

```toml
[tool.black]
line-length = 100
target-version = ["py311"]

[tool.isort]
profile = "black"
line_length = 100
multi_line_output = 3
include_trailing_comma = true

[tool.ruff]
line-length = 100
target-version = "py311"
exclude = ["venv", ".venv", "build", "dist", "migrations"]

[tool.ruff.lint]
# Базовые и расширенные правила: ошибки, предупреждения, импорт, безопасность
select = ["E", "F", "W", "I", "B", "S", "UP"]
ignore = ["E501", "B008", "S101", "S603", "S607"]

[tool.mypy]
python_version = "3.11"
ignore_missing_imports = true
warn_return_any = true
warn_unused_ignores = true
disallow_untyped_defs = true
```

---

## 5.3 Зависимости

**`requirements.txt`** (runtime):

```txt
fastapi==0.112.2
uvicorn==0.30.5
sqlalchemy==2.0.43
python-multipart==0.0.9
```

**`requirements-dev.txt`** (dev/test/quality):

```txt
pytest==8.2.2
httpx==0.27.2
ruff==0.6.9
black==24.8.0
isort==5.13.2
pre-commit==3.8.0
pytest-cov==5.0.0
mypy==1.11.1
bandit==1.7.9
```

---

## 5.4 Что проверяется и как это закрывает критерии ТМ 0.15

- **SAST‑контроли:** `bandit` (security) + `ruff` (включая правила `S*`) — автоматизированный анализ уязвимостей и небезопасных вызовов.  
- **Качество кода:** `ruff`, `black`, `isort` — единый стиль и чистые импорты.  
- **Типобезопасность:** `mypy` — запрет необъявленных/неаннотированных дефов.  
- **Надёжность и метрики:** `pytest` + `pytest-cov` с `--cov-fail-under=80` — выполняет **NFR‑08**.  
- **Репетируемость/скорость:** кэш pip, фиксированные версии, запуск по `push`/`pull_request`.  
- **Защита от дублей:** `concurrency` предотвращает параллельные гонки одного коммита.

---

## 5.5 Как прогнать локально (репродукция CI)

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt -r requirements-dev.txt
pre-commit install

ruff check --fix .
black .
isort .
mypy app
bandit -r app -ll
pytest --cov=app --cov-report=term-missing
```

## 6. Скриншоты работы

<img width="592" height="61" alt="Снимок экрана 2025-11-02 в 23 27 43" src="https://github.com/user-attachments/assets/21064e4d-a24a-4e59-9595-1e0b209ef475" />

<img width="421" height="140" alt="Снимок экрана 2025-11-02 в 23 29 26" src="https://github.com/user-attachments/assets/b4ae9cb3-db25-4db4-be0e-e41bf0fb4bea" />

<img width="425" height="101" alt="Снимок экрана 2025-11-02 в 23 29 51" src="https://github.com/user-attachments/assets/096364bf-eb1a-4ef6-a266-d9fe56f6f65d" />

<img width="810" height="618" alt="Снимок экрана 2025-11-02 в 23 30 20" src="https://github.com/user-attachments/assets/265157ff-6265-4b5e-965e-dcc80c6eb900" />

<img width="836" height="551" alt="Снимок экрана 2025-11-02 в 23 30 38" src="https://github.com/user-attachments/assets/6dc88d3b-1b5a-4c98-8596-083949fb74f7" />

<img width="711" height="588" alt="Снимок экрана 2025-11-02 в 23 30 55" src="https://github.com/user-attachments/assets/64cdc422-2a46-4d68-a95b-8885a3cb209c" />

