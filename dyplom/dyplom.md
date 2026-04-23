- 1) Минимальный read-API поверх уже собранных данных (чтобы и дашборд, и бот работали через один контракт): latest run, список run’ов, снапшоты по `source`, простые фильтры по датам.
- 2) Планировщик сбора: Celery + Redis + beat, таск “collect_all → persist”, плюс идемпотентность/дедуп (чтобы не плодить дубли).
- 3) Нормализованный “доменный слой” поверх raw JSON: таблицы/таймсерии для TVL/APY/flows и маппинг DeFiLlama → ваши сущности. Без этого аналитика/сравнения/скоринг будут болью.
- 4) Telegram bot “тонким срезом”: `/latest`, “top yields”, подписки, ежедневный дайджест — читает из БД (не дергает источники онлайн).
- 5) Risk scoring + explainability: сначала простой rule-based скоринг + объяснение факторов, потом ML.

# Команды для запуска проекта на данном этапе
#### 3) Применить миграции (создать таблицы)

python backend/manage.py migrate

#### 4) Собрать raw данные DeFiLlama и записать в Postgres (без JSON файлов)

python -m backend.data_collectors --no-save

#### 5) Нормализовать raw → канонические таблицы (с учётом allowlist)

python backend/manage.py normalize_defillama

#### 6) (Опционально) Запустить веб-сервис для проверки

python backend/manage.py runserver

# Как идет сбор данных в проекте и их нормализация под RWA

### Сбор и нормализация данных (текущее состояние проекта)

#### 0) Подготовка (один раз)

- В `.env` задан `DATABASE_URL` (Postgres).
- Выполнены миграции:

python backend/manage.py migrate

После этого в БД есть таблицы raw-слоя (`collection_runs`, `source_snapshots`) и канонического слоя (`protocols`, `protocol_metric_points`, `yield_pools`, `yield_pool_metric_points`).

---

## 1) Сбор данных (raw layer)

### Команда

python -m backend.data_collectors --no-save

### Что происходит по шагам

- Точка входа: `backend/data_collectors/__main__.py`
    
    - читает `.env` через `load_dotenv()`
    - поднимает настройки через `load_collector_settings()`
- Оркестратор: `backend/data_collectors/service.py`
    
    - создаёт `CollectionBundle(collected_at_utc=..., meta=...)`
    - параллельно делает 2 HTTP запроса:
        - `defillama_protocols`: `GET https://api.llama.fi/protocols`
        - `defillama_yields`: `GET https://yields.llama.fi/pools`
    - собирает результаты в `bundle.sources[]` (каждый source содержит `ok/error/data`)
- Запись в Postgres: `backend/core/db.py::persist_collection_bundle()`
    
    - вызывает `_ensure_django()` (ставит `DJANGO_SETTINGS_MODULE`, делает `django.setup()`)
    - создаёт одну запись `CollectionRun` (1 запуск сбора)
    - создаёт несколько записей `SourceSnapshotRow` (по 1 на источник)
        - `source`: например `defillama_protocols`, `defillama_yields`
        - `ok/error`
        - `data`: JSON без ключей `source/ok/error` (только payload)

### Итог raw-слоя

- `collection_runs`: 1 строка на запуск
- `source_snapshots`: 1+ строк на запуск (по источникам)

Raw — это “источник правды”, который можно пересчитывать в нормализованные метрики.

---

## 2) Нормализация данных (canonical/normalized layer)

### Команда

python backend/manage.py normalize_defillama

### Что происходит по шагам

- Команда: `backend/core/management/commands/normalize_defillama.py`
    
    - берёт последний `CollectionRun` (или `--run-id`)
    - вызывает `normalize_defillama_run(run_id=...)`
- ETL-логика: `backend/core/normalization/defillama.py`
    
    - читает raw снапшоты из `source_snapshots` для этого run:
        - `defillama_protocols` (список протоколов)
        - `defillama_yields` (список yield pools)
    - timestamp для метрик берётся из `CollectionRun.collected_at_utc`
    - allowlist фильтрация (опционально):
        - `RWA_PROTOCOL_SLUGS` → какие `protocol.slug` считать RWA
        - `RWA_YIELD_PROJECTS` → какие `pool.project` считать RWA
        - если переменные не заданы → фильтрации нет

### Во что нормализуем

- Протоколы:
    - `protocols` (справочник)
    - `protocol_metric_points` (точки метрик во времени, минимум: `tvl_usd`)
- Yield pools:
    - `yield_pools` (справочник пулов)
    - `yield_pool_metric_points` (точки метрик во времени: `apy`, `tvl_usd`)

Запись идемпотентная:

- на `(entity, ts_utc)` стоит уникальность
- повторная нормализация того же run обновит значения на этом timestamp, а не создаст дубликаты

### Итог нормализованного слоя

- Быстрые запросы для Telegram/дашборда:
    - “top TVL протоколы”
    - “top APY pools”
    - “график TVL/APY за период”
- Raw остаётся для дебага/пересчёта/аудита источника.

### 3) Где это потом используется (для Telegram/AI/дашборда)

#### Что считать “источником правды”

- `source_snapshots` — источник правды (raw).  
    Если что-то “странно” в метриках — мы всегда можем открыть raw JSON и понять, что реально вернул DeFiLlama.

#### Что использовать в продукте

- Нормализованные таблицы — то, что читают:
    - Telegram bot (топы, деталка)
    - будущий dashboard (графики)
    - AI agent (факты для ответов)

Почему: быстрые SQL-запросы, индексы, удобные выборки по времени.

---

### 4) Ключевые файлы и их роли (шпаргалка)

- Сбор
    - `backend/data_collectors/__main__.py` — CLI вход
    - `backend/data_collectors/service.py` — оркестрация шагов + запись raw
    - `backend/data_collectors/sources/defillama.py` — HTTP вызовы DeFiLlama
    
- Запись raw в Postgres
    - `backend/core/db.py` — `persist_collection_bundle()` (+ `django.setup()` чтобы ORM работал вне `manage.py`)
       
- Нормализация
    - `backend/core/management/commands/normalize_defillama.py` — CLI команда
    - `backend/core/normalization/defillama.py` — ETL raw → canonical
    
- Схема БД
    - `backend/core/models.py` — модели ORM (raw + canonical)
    - `backend/core/migrations/*` — миграции, создающие таблицы
    
---

### 5) Как контролировать “только RWA” (allowlist)

#### Переменные окружения

- `RWA_PROTOCOL_SLUGS` — фильтр для `defillama_protocols`
- `RWA_YIELD_PROJECTS` — фильтр для `defillama_yields`

#### Эффект

- Raw собирается как есть (может быть всё)
- В канонические таблицы попадёт только то, что в allowlist  
    → Telegram/AI/дашборд будут видеть только RWA.

---

### 6) Минимальная схема автоматизации (как будет “в проде”)

Пока вручную:

1. `python -m backend.data_collectors --no-save`
2. `python backend/manage.py normalize_defillama`

Потом автоматизация двумя задачами (Celery Beat или cron):

- collect job → пишет raw
- normalize job → обновляет canonical