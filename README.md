# Сервис НСИ (Master Data Service)

Микросервис для управления нормативно-справочной информацией (НСИ): справочники, таксономии, организационные иерархии и маршруты. Предоставляет унифицированный доступ к эталонным данным через HTTP REST API и RabbitMQ.

## Технологический стек

- **Язык:** Rust (Edition 2021)
- **Web-фреймворк:** Actix-web
- **База данных:** PostgreSQL (SQLx с компайл-тайм проверкой запросов)
- **Брокер сообщений:** RabbitMQ (amqprs)
- **Async runtime:** Tokio
- **Логирование:** CEF-формат через tracing

## Требования

- Rust 1.57.0+
- PostgreSQL 12+
- RabbitMQ 3.8+

## Быстрый старт

### Локальный запуск

```bash
# Клонировать репозиторий
git clone https://github.com/MShubkin/master-data-service.git
cd master-data-service

# Настроить переменные окружения
cp .env .env.local
# Отредактировать .env.local под своё окружение

# Собрать и запустить
cargo run
```

Сервис запустится на `http://0.0.0.0:9071`.

### Docker

```bash
# Сборка образа
docker build \
  --build-arg RUST_VERSION=1.57.0 \
  --build-arg GIT_COMMIT_ID=$(git rev-parse HEAD) \
  --build-arg BUILD_VERSION_ID=1.0.0 \
  -t master-data-service:latest .

# Запуск контейнера
docker run -p 9071:9071 \
  -e POSTGRES_VHOST=postgres-host \
  -e RABBITMQ_HOST=rabbitmq-host \
  master-data-service:latest
```

## Конфигурация

Все параметры задаются через переменные окружения (файл `.env`):

| Переменная | По умолчанию | Описание |
|---|---|---|
| `SRV_HOST` | `0.0.0.0` | Адрес для привязки сервиса |
| `SRV_PORT` | `9071` | Порт сервиса |
| `SRV_THREAD_COUNT` | `7` | Размер пула потоков |
| `LOGGER_MODE` | `cef` | Режим логирования (`cef` / `dev` / `none`) |
| `LOGGER_DIR` | `logs` | Директория логов |
| `RABBITMQ_HOST` | `localhost` | Хост RabbitMQ |
| `RABBITMQ_PORT` | `5672` | Порт RabbitMQ |
| `RABBITMQ_VHOST` | `/` | Virtual host RabbitMQ |
| `RABBITMQ_USERNAME` | `guest` | Пользователь RabbitMQ |
| `RABBITMQ_RETRIES` | `10` | Количество попыток подключения |
| `POSTGRES_VHOST` | `localhost` | Хост PostgreSQL |
| `POSTGRES_PORT` | `5432` | Порт PostgreSQL |
| `POSTGRES_DB` | `postgres` | Имя базы данных |
| `POSTGRES_USER` | `postgres` | Пользователь БД |
| `POSTGRES_PASSWORD` | `postgres` | Пароль БД |
| `POSTGRES_MIN_CONNECTIONS` | `1` | Минимум соединений в пуле |
| `POSTGRES_MAX_CONNECTIONS` | `4` | Максимум соединений в пуле |
| `MONOLITH_BASE_URL` | `http://localhost:8080` | URL монолитного сервиса |
| `MASTER_DATA_BASE_URL` | `http://localhost:9071` | URL этого сервиса |
| `RUST_LOG` | `debug` | Уровень логирования |

## HTTP API

Базовый URL: `http://{host}:9071`

### Мониторинг

| Метод | Путь | Описание |
|---|---|---|
| `GET` | `/monitoring/test` | Проверка доступности сервиса |
| `GET` | `/monitoring/config` | Конфигурация сервера |

### Справочники

| Метод | Путь | Описание |
|---|---|---|
| `POST` | `/v1/{directory}/search/` | Полнотекстовый поиск в справочнике |
| `POST` | `/v1/{directory}/search_by_id/` | Поиск по списку ID |
| `GET` | `/v1/{directory}/{id}/` | Получить запись по ID |
| `GET` | `/v1/master_data/get_updates/{timestamp}/` | Получить изменения с указанной метки времени |
| `GET` | `/v1/get/hierarchy/{dictionary}/` | Получить иерархическую структуру |

### Избранные справочники

| Метод | Путь | Описание |
|---|---|---|
| `GET` | `/v1/get/favorite_list/` | Список избранных справочников |
| `POST` | `/v1/create/favorite_item/` | Добавить в избранное |
| `DELETE` | `/v1/delete/favorite_item/` | Удалить из избранного |

### Организационная структура

| Метод | Путь | Описание |
|---|---|---|
| `POST` | `/v1/organizational_structure/search/` | Поиск организаций |
| `POST` | `/v1/organizational_structure/search_by_id/` | Найти организацию по ID |
| `POST` | `/v1/organizational_user_assignment/search/` | Поиск назначений пользователей |
| `POST` | `/v1/organizational_user_assignment/search_by_id/` | Найти назначение по ID |

### Основания отмены планов

| Метод | Путь | Описание |
|---|---|---|
| `POST` | `/v1/plan_reason_cancel/search/` | Поиск оснований |
| `POST` | `/v1/plan_reason_cancel/search_by_id/` | Найти по ID |
| `POST` | `/v1/plan_reason_cancel/get/item_list/` | Список оснований |
| `POST` | `/v1/plan_reason_cancel/get/detail/` | Детальная информация |
| `POST` | `/v1/plan_reason_cancel/create/item/` | Создать основание |
| `POST` | `/v1/plan_reason_cancel/update/item/` | Обновить основание |
| `POST` | `/v1/plan_reason_cancel/delete/item/` | Удалить основание |
| `POST` | `/v1/plan_reason_cancel/restore/item/` | Восстановить основание |
| `POST` | `/v1/plan_reason_cancel/export/` | Экспорт |

### Маршруты

| Метод | Путь | Описание |
|---|---|---|
| `POST` | `/v1/test/routes/` | Тестирование подбора маршрута |

## RabbitMQ

### Очереди

| Очередь | Переменная окружения | Описание |
|---|---|---|
| Dictionary queue | `REQUEST_DICTIONARY_QUEUE` | Запросы на получение справочников |
| Action queue | `MASTER_DATA_ACTION_QUEUE` | Операции с мастер-данными |
| Routing queue | `ROUTING_QUEUE` | Подбор и автоназначение маршрутов |

### Поддерживаемые действия (Action queue)

- `RouteCreate` / `RouteUpdate` / `RouteRemove` / `RouteCopy` — CRUD маршрутов
- `RouteStart` / `RouteStop` — управление состоянием маршрута
- `RouteList` / `RouteDetails` / `RouteFind` — получение маршрутов
- `OrganizationUserAssignmentSearchById` / `OrganizationUserAssignmentSearchByDepartment` — назначения пользователей
- `SearchPlanReasonCancel` — поиск оснований отмены планов

## Справочники НСИ

При запуске сервис загружает все справочники в оперативную память для быстрого доступа:

| Ключ | Описание |
|---|---|
| `budget_item` | Статьи бюджета |
| `category` | Категории товаров/услуг |
| `okpd2` | ОКПД2 — классификатор видов деятельности |
| `organization` | Поставщики/организации |
| `organizational_structure` | Организационная иерархия |
| `object_type` | Типы объектов |
| `expert_conclusion_type` | Типы экспертных заключений |
| `critical_type_color_scheme` | Цветовые схемы критичности |
| `agenda_status` | Статусы повестки комиссии |
| `estimated_commission_result` | Решения комиссии СК |
| `estimated_commission_protocol_status` | Статусы протокола |
| `protocol_type` | Типы протоколов |
| `estimated_commission_role` | Роли членов комиссии |
| `price_information_request_type` | Типы запросов ценовой информации |
| `assigning_executor_method` | Методы назначения исполнителя |
| `analysis_method` | Методы анализа |
| `price_analysis_method` | Методы анализа цен |
| `pricing_organization_unit` | Подразделения ценообразования |
| `payment_conditions` | Условия оплаты |
| `ppz_type` | Типы планов размещения |
| `response` | Ответы подразделений |
| `attachment_type` | Типы вложений |
| `output_form` | Форматы выходных документов |
| `technical_commercial_proposal_status` | Статусы ТКП |
| `scheduler_request_update_catalog` | Производственные календари |
| `plan_reasons_cancel_impact_area` | Области влияния отмены плана |
| `plan_reasons_cancel_functionality` | Функциональность отмены плана |
| `plan_reasons_cancel_check_reason` | Основания проверки отмены |
| `favorite_dictionaries` | Избранные справочники пользователя |

## Архитектура

Проект построен по принципам чистой архитектуры:

```
src/
├── application/      # Бизнес-логика: действия над маршрутами, поиск, обновления
│   ├── action/       # Обработчики действий (RouteCreate, RouteFind и др.)
│   └── master_data/  # Загрузка и управление справочниками
├── domain/           # Доменные интерфейсы (MasterDataRecord, MasterDataDirectory)
├── infrastructure/   # Внешние сервисы: БД, RabbitMQ, HTTP-сервер, конфигурация
└── presentation/     # HTTP-обработчики и RabbitMQ-обработчики, DTO
```

**Ключевые особенности реализации:**
- Все справочники загружаются при старте в `OnceCell<MasterData>` — O(1) доступ во время работы
- SQLx обеспечивает проверку SQL-запросов на этапе компиляции
- Полностью асинхронный код на Tokio
- Двойной интерфейс: HTTP REST + RabbitMQ

## Разработка

```bash
# Запуск тестов
cargo test

# С интеграционными тестами
cargo test --features integration_tests

# Сборка релизной версии
cargo build --release
```

