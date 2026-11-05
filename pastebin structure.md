# Общая структура проекта (Project Structure)

Проект организован по принципам чистой микросервисной архитектуры на базе единого `.sln` решения C# (.NET 8) с готовой конфигурацией для локального запуска в `docker-compose` и деплоя в кластер **Kubernetes**.

## 1. Дерево директорий репозитория

```
pastebin-system/
├── README.md
├── ARCHITECTURE.md
├── PROJECT_STRUCTURE.md
├── Pastebin.sln                         # Решение .NET
├── docker-compose.yml                   # Локальное окружение (Postgres, Redis, MinIO, Сервисы)
├── .env.example                         # Пример переменных окружения
│
├── src/                                 # Исходный код сервисов
│   ├── Pastebin.Shared/                 # Общие контракты, gRPC proto-файлы, утилиты
│   │   ├── Protos/
│   │   │   └── hash_generator.proto     # gRPC контракт между MainService и HashService
│   │   ├── Constants/
│   │   │   └── TtlPrefixes.cs           # Константы префиксов S3 (ttl-1h, ttl-1d, ttl-7d)
│   │   └── Pastebin.Shared.csproj
│   │
│   ├── Pastebin.MainService/            # Основной сервис (Backend + простой Frontend)
│   │   ├── Dockerfile
│   │   ├── Program.cs                   # Конфигурация DI, Kestrel (лимит 10 МБ), пайплайна
│   │   ├── appsettings.json
│   │   ├── Endpoints/
│   │   │   └── PasteEndpoints.cs        # POST /api/pastes, GET /{hash}, редиректы
│   │   ├── Services/
│   │   │   ├── IPasteOrchestrator.cs    # Логика синхронного POST и кэширующего GET
│   │   │   ├── PasteOrchestrator.cs
│   │   │   ├── S3StorageService.cs      # Работа с Amazon S3 SDK (загрузка и чтение потоков)
│   │   │   └── RedisCacheService.cs     # Работа с кэшем частых хэшей и кэшем текстов
│   │   ├── Data/
│   │   │   ├── PastebinDbContext.cs     # EF Core / Dapper контекст для PostgreSQL
│   │   │   ├── Entities/
│   │   │   │   └── PasteMetadata.cs     # Сущность таблицы pastes
│   │   │   └── Migrations/              # Миграции схемы БД
│   │   ├── wwwroot/                     # Простой фронтенд (сбор и отображение текста)
│   │   │   ├── index.html               # Форма отправки текста и выбора TTL
│   │   │   ├── view.html                # Страница отображения текста по /hash_code
│   │   │   └── styles.css
│   │   └── Pastebin.MainService.csproj
│   │
│   ├── Pastebin.HashService/            # Сервис генерации 8-значных Base62 хэшей
│   │   ├── Dockerfile
│   │   ├── Program.cs
│   │   ├── appsettings.json
│   │   ├── GrpcServices/
│   │   │   └── HashGrpcService.cs       # Выдача хэшей основному сервису
│   │   ├── Core/
│   │   │   ├── Base62Generator.cs       # Алгоритм генерации 8-значных строк Base62
│   │   │   └── HashPoolManager.cs       # Потокобезопасное получение хэшей из Redis
│   │   ├── BackgroundWorkers/
│   │   │   └── PoolReplenishWorker.cs   # Фоновый мониторинг и догенерация хэшей в кэш
│   │   └── Pastebin.HashService.csproj
│   │
│   └── Pastebin.CleanupWorker/          # Воркер удаления истекших записей из PostgreSQL
│       ├── Dockerfile
│       ├── Program.cs
│       ├── appsettings.json
│       ├── Workers/
│       │   └── ExpiredPastesCleaner.cs  # Фоновый цикл очистки таблицы pastes по expires_at
│       └── Pastebin.CleanupWorker.csproj
│
├── tests/                               # Модульные и интеграционные тесты
│   ├── Pastebin.UnitTests/              # Тесты Base62, оркестратора и валидации
│   └── Pastebin.IntegrationTests/       # Тесты пайплайна с Testcontainers (Postgres, Redis, LocalStack)
│
└── deploy/                              # Инфраструктура и деплой
    ├── s3/
    │   └── lifecycle-policy.json        # Правила удаления объектов S3 по префиксам TTL
    └── k8s/                             # Манифесты Kubernetes
        ├── namespace.yaml
        ├── configmap.yaml
        ├── secrets.yaml
        ├── ingress.yaml                 # NGINX Ingress с лимитом body-size 10m
        ├── main-service-deployment.yaml
        ├── main-service-service.yaml
        ├── main-service-hpa.yaml        # Автоскейлинг основного сервиса
        ├── hash-service-deployment.yaml
        ├── hash-service-service.yaml
        ├── cleanup-worker-deployment.yaml
        ├── postgres-statefulset.yaml    # Или подключение к Managed PostgreSQL (AWS RDS)
        └── redis-statefulset.yaml       # Или подключение к Managed Redis (ElastiCache)

```

## 2. Описание проектов внутри решения (.sln)

### 2.1. `Pastebin.MainService`

Центральное звено системы.

* **Kestrel Configuration:** Настроен жесткий лимит `MaxRequestBodySize = 10 * 1024 * 1024` ($10\text{ МБ}$), чтобы отсекать слишком большие запросы на уровне веб-сервера до выделения памяти.

* **Фронтенд (`wwwroot`):** Состоит из двух минималистичных страниц:

  1. `GET /` — отображает текстовое поле (`<textarea>`), выпадающий список времени жизни (`1 час`, `1 день`, `7 дней`) и кнопку «Создать». Отправляет форму через `POST /api/paste`.

  2. `GET /{hash}` — запрашивает текст у бэкенда и рендерит его внутри блока `<pre><code>`.

* **Работа с кэшами Redis:** Использует библиотеку `StackExchange.Redis`. Поддерживает логическое разделение кэшей (например, через префиксы `meta:{hash}` и `text:{s3_key}` или разные логические базы данных Redis `db0`, `db1`).

### 2.2. `Pastebin.HashService`

Изолированный микросервис генерации ключей.

* **Интерфейс:** Взаимодействует с `MainService` по протоколу **gRPC** внутри кластера Kubernetes для минимальной задержки.

* **Многопоточность:**

  * При запросе `GetNextHash` вызывает атомарный `SPOP` из Redis-множества `hash_pool`.

  * `PoolReplenishWorker` периодически проверяет количество доступных хэшей в Redis. Если их меньше порогового значения (`MinPoolSize = 10000`), воркер в несколько потоков (`Parallel.For` / `RandomNumberGenerator`) генерирует пачку новых 8-значных Base62 строк и атомарно пушит их в Redis (`SADD`).

### 2.3. `Pastebin.CleanupWorker`

Легковесный сервис без открытых HTTP-портов (Worker Service).

* Содержит `PeriodicTimer`, который срабатывает каждые $N$ секунд.

* Выполняет батчевое удаление строк из PostgreSQL, где `expires_at <= NOW()`.

## 3. Конфигурация контейнеров и Kubernetes (`deploy/k8s/`)

### 3.1. Особенности Ingress-контроллера (`ingress.yaml`)

Поскольку система принимает текстовые блоки до $10\text{ МБ}$, стандартный лимит NGINX ($1\text{ МБ}$) приведет к ошибке `413 Payload Too Large`. В манифесте Ingress обязательно задаются аннотации:

```
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/limit-rps: "20" # Базовая защита от спама без авторизации

```

### 3.2. Масштабирование сервисов в Kubernetes

* **`main-service-deployment`:** Запускается от 2 и более реплик. Полностью stateless. Настраивается `HorizontalPodAutoscaler` (HPA) по метрикам CPU/Memory.

* **`hash-service-deployment`:** Запускается в 2 репликах для отказоустойчивости. Потокобезопасность между репликами обеспечивается атомарностью операций Redis и распределенной блокировкой (Redlock) на этапе фонового пополнения пула.

* **`cleanup-worker-deployment`:** Запускается в 1 реплике (`replicas: 1`), либо использует `FOR UPDATE SKIP LOCKED` в SQL-запросе, что позволяет безопасно запускать даже несколько реплик воркера параллельно.

### 3.3. Конфигурация правил жизненного цикла S3 (`deploy/s3/lifecycle-policy.json`)

Конфигурация бакета Amazon S3 для автоматического удаления файлов по префиксам:

```
{
  "Rules": [
    {
      "ID": "Expire-1h-Prefix",
      "Filter": { "Prefix": "ttl-1h/" },
      "Status": "Enabled",
      "Expiration": { "Days": 1 }
    },
    {
      "ID": "Expire-1d-Prefix",
      "Filter": { "Prefix": "ttl-1d/" },
      "Status": "Enabled",
      "Expiration": { "Days": 1 }
    },
    {
      "ID": "Expire-7d-Prefix",
      "Filter": { "Prefix": "ttl-7d/" },
      "Status": "Enabled",
      "Expiration": { "Days": 7 }
    }
  ]
}

```