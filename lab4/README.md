# "Разработка инфраструктуры MVP AI приложения Виноградов М. U4225
# Цель - Создать прототип AI-приложения с базовой функциональностью.

### Возьмем теоретический сервис для AI-проверки решений задач по математике. Пользователь фотографирует решение, сервис распознаёт его и через LLM возвращает разбор ошибок и оценку. Есть веб-клиент, личный кабинет, история проверок, кабинет куратора/школы.

## Таким, образом у нас будут следующие Исходные данные
| Параметр | 1. Начальное (MVP) | 2. Тест партнёрами | 3. Прод |
|---|---|---|---|
| Пользователи | команда и тестеры | 3–5 школ-партнёров, ~1 000 MAU | ~50 000 MAU, ~5 000 DAU |
| Проверок (LLM-запросов) | ~500/день, 15k/мес | ~5k/день, 150k/мес | ~100k/день, 3M/мес |
| Пиковая нагрузка | < 1 RPS | ~5–10 RPS | ~50–100 RPS |
| Хранилище фото | ~5 ГБ | ~50 ГБ | ~1 ТБ+ |
| Требования к доступности | не нужны | рабочее время | бэкапы, защита |
| Длительность этапа | ~2 мес | ~3–4 мес | постоянно дальше |

### Регион возьмем europe-north1. Запросов к LLM -- ~1 500 входных токенов (фото + промпт) и ~500 выходных

### С первого дня выбираем serverless + managed-сервисы, которые на малой нагрузке стоят почти ноль и растут изменением параметров без смены архитектуры

## Архитектура по этапам

### 1. Начальное решение
- Firebase Hosting / Cloud Storage — статический фронтенд
- Cloud Run api (FastAPI), min 0 / max 2, 1 vCPU, 512 MiB
- Vertex AI Gemini 2.5 Flash — проверка решений (синхронно, запросов мало)
- Cloud SQL PostgreSQL db-f1-micro, 10 ГБ, без HA
- Cloud Storage — фото решений
- Secret Manager — ключи; Artifact Registry + Cloud Build — CI/CD
- Cloud Logging / Monitoring — в рамках бесплатных квот

<img width="855" height="699" alt="image" src="https://github.com/user-attachments/assets/f5e5eb39-c224-41a4-8479-a9a351a1c733" />

### Вот такая схема инфраструктуры в draw io для первого этапа

### 2. Этап тестирования партнерами

- api: min instances 1 (у партнёров не должно быть cold start на демо)
- Pub/Sub + Cloud Run worker: проверка становится асинхронной (очередь), чтобы пик запросов не упирался в лимиты LLM и таймауты HTTP
- Identity Platform — вход для учеников и кураторов школ (бесплатно до 50k MAU)
- Cloud SQL → db-g1-small, 20 ГБ, автоматические бэкапы + PITR
- Отдельный проект staging (копия в минимальной конфигурации) — партнёры работают на стабильном окружении, разработка не ломает им прод
- Алерты в Cloud Monitoring (ошибки 5xx, латентность, бюджет)

<img width="1280" height="682" alt="image" src="https://github.com/user-attachments/assets/33a48315-482f-47ea-aaf8-4360f4ddaf65" />

### А вот такая схема инфраструктуры в draw io для второго этапа

### 3. Этап продового решения

- External HTTPS Load Balancer + Cloud CDN + Cloud Armor — один домен, кеш статики и картинок, WAF и rate limiting
- api: min 2 / max 30; worker: min 1 / max 50
- Memorystore Redis — кеш сессий и повторных проверок (одинаковые задачи из одного домашнего задания)
- Cloud SQL → db-custom-2-7680 HA (regional) + read replica для аналитики
- Gemini 2.5 Flash-Lite для простых задач (маршрутизация по сложности) + Flash для сложных, context caching для общего промпта
- Cloud Storage lifecycle: фото старше 90 дней → Nearline
- BigQuery — аналитика для школ (выгрузка из Cloud SQL/Pub/Sub)
- Три проекта: dev / staging / prod; Committed Use Discounts на Cloud SQL (−25…52 %)

 <img width="1258" height="559" alt="image" src="https://github.com/user-attachments/assets/e6f0fea9-ecff-4ff9-b196-82d463bb4d99" />
 
### И вот такая схема инфраструктуры в draw io для третьего этапа


## По этапам у нас будет следующий состав сервисов:

| Компонент | Этап 1 | Этап 2 | Этап 3 |
|---|---|---|---|
| Фронтенд | Firebase Hosting | Firebase Hosting | Cloud Storage + Cloud CDN |
| Точка входа | URL Cloud Run | URL Cloud Run | HTTPS Load Balancer + Cloud Armor |
| API | Cloud Run, min 0 / max 2 | Cloud Run, min 1 / max 5 | Cloud Run, min 2 / max 30 |
| Обработка проверок | Синхронно в API | Pub/Sub + Cloud Run worker | Pub/Sub + Cloud Run worker, max 50 |
| LLM | Gemini 2.5 Flash | Gemini 2.5 Flash | Flash-Lite + Flash (роутинг), context caching |
| БД | Cloud SQL db-f1-micro | Cloud SQL db-g1-small, бэкапы + PITR | Cloud SQL HA 2 vCPU / 7.5 ГБ + read replica |
| Кеш | — | — | Memorystore Redis |
| Файлы | Cloud Storage Standard | Cloud Storage Standard | Standard → Nearline через 90 дней |
| Аутентификация | — | Identity Platform | Identity Platform |
| Аналитика | — | — | BigQuery |
| Окружения | одно | Prod + staging | Dev / staging / prod |
| Мониторинг | Базовый | Алерты на ошибки и бюджет | Алерты, SLO, дашборды |

## Экономическая модель

## Стоимость одной проверки (для Gemini 2.5 Flash, у которого ≈ $0.30 / 1M input, $2.50 / 1M output)
## будет примерно равна 0.0017 USD (а если взять Flash-Lite, то примерно 0.00035 USD)

## Каждый этап прогоняем через Google Cloud Pricing Calculator и получаем следующую экономическую модель:

| Статья расходов, USD/мес | Этап 1 | Этап 2 | Этап 3 |
|---|---:|---:|---:|
| Cloud Run (api + worker) | 0, БЕСПЛАТНО | ~25 | ~350 |
| Cloud SQL PostgreSQL | ~11 (db-f1-micro) | ~30 (db-g1-small + бэкапы) | ~330 (HA + replica + 100 ГБ SSD) |
| Cloud Storage | < 1 | ~1 | ~20 (с lifecycle) |
| Memorystore Redis | — | — | ~60 |
| Load Balancer + Cloud Armor + CDN + egress | — | — | ~100 |
| Pub/Sub, Secret Manager, Artifact Registry | 0 | ~2 | ~15 |
| Cloud Logging / Monitoring | 0 | ~5 | ~50 |
| BigQuery | — | — | ~20 |
| Identity Platform | — | 0 (< 50k MAU) | ~0–30 |
| Окружения staging / dev | — | ~15 | ~80 |
| **Vertex AI Gemini** | **~25** | **~255** | **~2 700** |
| **Итого** | **≈ 40** | **≈ 335** | **≈ 3 700** |
| Доля LLM в расходах | ~60 % | ~76 % | ~73 % |
| Стоимость на 1 MAU | — | ~0.34 | ~0.07 |

## Расчёт LLM на этапе 3 составит примерно 2670 USD

## Обоснование выбора тех или иных ресурсов
| Решение | Плюсы | Минусы | Вердикт |
|---|---|---|---|
| Одна VM e2-small со всем внутри | ~13 USD/мес на этапе 1 | ручные бэкапы, нет масштабирования, единая точка отказа, переделка на этапе 2 | принято не брать, т.к. чем дальше по этапам, тем дороже |
| Compute Engine MIG + Load Balancer | полный контроль | администрирование ОС, оплата простоя | не берем, т.к. избыточно для MVP |
| GKE | гибкость, стандарт для крупных систем | кластер от ~70 USD/мес + нужен контроль девопса |в рамках 3 этапа не берем|
| **Cloud Run** | scale-to-zero, оплата за запросы, один контейнер на всех этапах | cold start (решается min instances), только stateless | взяли |
| Firestore | бесплатен на старте | NoSQL, неудобные отчёты, дорогая миграция | не берем |
| **Cloud SQL PostgreSQL** | реляционная модель, рост сменой tier, HA одной настройкой | ~10 USD/мес даже в простое | взяли |
| Своя LLM на GPU (L4) | нет платы за токены | ~500+ USD/мес в простое, MLOps, качество ниже | не взяли;( |
| **Vertex AI Gemini** | оплата за использование, без MLOps | основная статья расходов | взяли с роутингом |

