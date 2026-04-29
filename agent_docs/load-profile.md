# Профиль нагрузки и нефункциональные требования (NFR)

Документ описывает целевой профиль нагрузки, performance targets, стратегию кэширования и шардинга, observability-стек и подход к тестированию системы ESD ServiceDesk на горизонте 24 месяца. Дополняет [architecture.md](./architecture.md) (компонентная архитектура) и [data-model.md](./data-model.md) (физическая модель данных и шардинг сущностей).

---

## 1. Ожидаемая нагрузка на горизонте 24 месяца

### 1.1. Базовые предпосылки

Прогноз масштаба к концу 24-го месяца эксплуатации, использованный для capacity planning:

| Параметр | Значение | Комментарий |
|---|---|---|
| Подключённых объектов (МКД, ЖК, КП) | 5 000+ | Целевой KPI Фазы 5 «Национальное масштабирование» |
| Активных пользователей-жителей | 3 000 000+ | DAU/MAU будет уточняться по факту |
| Подключённых УК | 800–1 200 | По 4–6 объектов на УК (медиана) |
| Подключённых ПАК-устройств | 15 000–20 000 | 3–4 точки доступа на объект (подъезды, шлагбаумы, калитки) |
| Партнёров B2B (Yandex/OZON/WB и т. п.) | 8–12 | Поставщики курьерских заявок |
| Проходов через точки доступа в сутки | 12 000 000 | ~4 прохода/день/пользователь × 3 М пользователей |
| Пропусков (разовых/гостевых/курьерских), создаваемых в сутки | 2 100 000 | ~5/неделя/пользователь × 3 М ÷ 7 |
| Heartbeat-сообщений ПАК в сутки | 14 400 000 | 1 сообщение / 30 с / устройство × 5 000 устройств × 86 400 / 30 |
| ОСС-голосований одновременно активных | 50–200 | Зависит от сезона, концентрируется в осенние/весенние месяцы |

### 1.2. Усреднённый RPS

Средние значения (uniform distribution), без учёта суточных пиков:

- **Проходы (access events):** 12 М / 86 400 с ≈ **140 RPS** (запись в `access_events` + ACL-проверка).
- **Создание пропусков:** 2,1 М / 86 400 с ≈ **24 RPS**.
- **Heartbeat ПАК:** 14,4 М / 86 400 с ≈ **170 RPS постоянный**.
- **REST-вызовы кабинета жителя/УК:** ~50 RPS среднее (зависит от MAU и сессий).

### 1.3. Пиковые сценарии

| Сценарий | Время | Целевая подсистема | Пик RPS | Особенности |
|---|---|---|---|---|
| Утренний выход на работу | 08:00–09:00 МСК | `acl-svc`, `device-gateway` | до **1 400 RPS** на ACL-проверки (10× от среднего) | BLE-критично: пользователь стоит у двери, p99 latency `acl_check` должен быть < 100 ms |
| Дневная доставка / обед | 12:00–13:00 МСК | `partners-svc` (Yandex GO, OZON Express, WB-логистика) | до **2 000 RPS** на выдачу partner one-time keys | Курьеры с агрегированным трафиком от B2B-партнёров |
| Вечерняя доставка | 18:00–20:00 МСК | `partners-svc` | до **3 000 RPS** на выдачу partner keys | Самый нагруженный пик; параллельно растёт ACL-нагрузка |
| Финал ОСС-голосования | последние 24 ч голосования | `oss-svc` + `audit-svc` | до **500 RPS** на запись голосов | CPU-bound: XAdES-T verify через КриптоПро (~30–80 ms на проверку), требует пула воркеров и HSM |
| Heartbeat-стрим ПАК | постоянно 24/7 | `device-gateway`, TimescaleDB | **170 RPS sustained** | Желательно через NATS JetStream (low-latency), а не Kafka |
| Партнёрские webhooks (callback на события) | следует за пиками доставок | `notifications-svc` consumers | до **300 RPS burst**, **140 RPS среднее** | Идемпотентность через ключ `partner_order_id`, ретраи с exp backoff |
| Поиск пропусков жителем (UI) | равномерно | `passes-svc` через Redis | ~**50 RPS среднее**, до **200 RPS burst** | Кэш на 30 с (см. § 4) |
| Запросы к AI-ассистенту | концентрация в рабочие часы УК | `assistant-svc` (Claude + Qdrant) | ~**5–15 RPS** | Латентность 2–6 с (LLM-bound), не ACL-критично |
| Push-уведомления о пропусках | в момент создания + при изменении статуса | `notifications-svc` consumers | до **500 RPS burst** | Сжатие в батчи на 100 ms окна |

---

## 2. Performance targets (SLO)

| Подсистема | Целевая метрика | Значение |
|---|---|---|
| API Gateway (Kong) | RPS sustained | **5 000 RPS** |
| API Gateway | RPS burst (≤ 60 с) | **15 000 RPS** |
| API Gateway | p99 latency без бэкенда | < 10 ms |
| `acl-svc` (Go) | p99 latency ACL-проверки | **< 100 ms** (BLE-критично) |
| `acl-svc` | p99 latency при cache hit | < 20 ms |
| `passes-svc` (Python/FastAPI) | p95 latency `POST /passes` | < 250 ms |
| `passes-svc` | p95 latency `GET /passes` (кэшированный) | < 80 ms |
| `partners-svc` (Go) | p99 latency `POST /b2b/v1/keys/issue` | < 200 ms |
| `oss-svc` | p95 latency `POST /oss/{id}/vote` (с XAdES-T) | < 600 ms |
| ClickHouse (`access_events`) | inserts/sec capacity | **20 000 inserts/sec** |
| ClickHouse | p99 query latency для отчётов УК | < 2 с (на горизонт 30 дней по одному объекту) |
| PostgreSQL Citus | TPS write (mixed insert/update) | **5 000 TPS** |
| PostgreSQL Citus | p99 read latency (по shard key) | < 30 ms |
| Redis Cluster | ops/sec | **50 000 ops/sec** |
| Redis Cluster | p99 GET/SET latency | < 2 ms |
| Kafka | throughput | **100 МБ/сек** |
| Kafka | конфигурация | 3 топика по 30 партиций (типовой топик), retention 7 дней (горячий), 30 дней (audit), компактация для compacted-топиков (`device.config`, `partner.quota`) |
| NATS JetStream (BLE events) | p99 publish→consume | < 50 ms |
| Доступность системы (composite SLA) | uptime | **99,9 % / месяц** (≤ 43,2 мин downtime) |
| RPO (Recovery Point Objective) | потеря данных при катастрофе | ≤ 5 мин (горячая репликация PG-Citus и ClickHouse) |
| RTO (Recovery Time Objective) | восстановление в DR-регионе | ≤ 60 мин |

---

## 3. Стратегия кэширования (Redis Cluster)

Кэш — критический элемент для достижения p99 < 100 ms на ACL и < 80 ms на «горячих» REST-запросах. Все ключи имеют префикс namespace, инвалидация — через Pub/Sub-канал `cache:invalidate` либо через Outbox-events из основной БД.

| Ключ кэша | Содержимое | TTL | Инвалидация |
|---|---|---|---|
| `pass:{pass_id}:valid_points` | Список AccessPoint, к которым валиден пропуск, и срок | 5 мин | Pub/Sub-событие `pass.revoked` / `pass.expired` (через Kafka → Redis publisher) |
| `acl:{user_id}:{access_point_id}` | Решение `granted` / `denied` + reason для частых проверок | 60 с | Pub/Sub при изменении `Pass`, `Resident.role`, `Subscription.status` |
| `user:{user_id}:passes` | Список активных пропусков жителя для UI | 30 с | Outbox-event из `passes-svc` при CREATE/UPDATE/REVOKE |
| `object:{object_id}:profile` | Объект + точки доступа + текущая УК + субподписка | 60 мин | При изменении объекта/точек доступа или смене УК |
| `uk:{uk_id}:profile` | УК + лицензия 191-ФЗ + договор с ESD + текущий уровень подписки | 60 мин | При изменении договора или подписки |
| `esia:session:{token_hash}` | userinfo от ЕСИА (минимизированный профиль) | 15 мин | По logout или по `expires_in` от ЕСИА |
| `partner:{partner_id}:quota:{YYYYMM}` | Использованная квота B2B-ключей в месяце | до конца месяца | Atomic INCR при выдаче ключа; обнуление через cron 1-го числа |
| `device:{serial}:config` | Текущий desired-config устройства (для polling) | 5 мин | По apply конфигурации в `device-svc` |
| `device:{serial}:keys_set` | Активный набор ключей для оффлайн-валидации | 5 мин | При revoke/issue любого ключа на объекте |
| `gis_gkh:nsi:{dictionary}` | Справочники ГИС ЖКХ (ОКТМО, регионы, виды работ) | 24 ч | Cron-обновление 1×/сутки через `/NsiService/` |
| `fias:address:{guid}` | Резолв ФИАС GUID → нормализованный адрес | 7 дней | По manual-синхронизации |
| `oss:{oss_id}:results` | Промежуточные результаты голосования | 30 с | Atomic INCR для голосов, итог пишется по завершении |
| `lock:resource:{resource_id}` | Распределённый лок (RedLock) на одновременную операцию | 30 с | По завершении операции |
| `rate_limit:{partner_id}:{endpoint}` | Sliding-window rate limit для партнёров | 1 мин | Atomic INCR + EXPIRE |

Дополнительно используются **Redis Streams** для очередей коротких задач (notifications fan-out, OTP-codes), где не требуется durability Kafka.

---

## 4. Шардинг и георезервирование

### 4.1. PostgreSQL (Citus)

- **Топология:** 1 coordinator + **4 worker-ноды** в primary-регионе (Москва), все на Citus 12+ поверх PostgreSQL 16.
- **Ключ шардирования:** `object_id` для всех таблиц, привязанных к объекту (`pass`, `key`, `oss`, `oss_vote`, `subscription`, `installer_job`, `audit_log`). Глобальные таблицы (`uk`, `object`, `device`, `user`, `partner`) — reference-таблицы, реплицируемые на каждый worker.
- **Number of shards:** 64 шарда на каждую распределённую таблицу (запас на 4× рост без re-balance).
- **Реплики:** каждый worker имеет hot standby в той же AZ (synchronous, для RPO=0 на горячую транзакцию) + asynchronous replica в другой AZ (RPO ≤ 5 с).
- **Cross-AZ:** primary-coordinator + workers распределены между 2 AZ внутри Москвы.
- **DR:** cold-replica в DR-регионе (Санкт-Петербург или Екатеринбург) с lag ≤ 5 мин.
- **Connection pooling:** PgBouncer перед каждым worker (transaction-mode), PgCat — перед coordinator.
- **Backup:** WAL-G в S3 RuStack (VK Cloud / Yandex Object Storage), PITR с глубиной 30 дней.

### 4.2. ClickHouse (журналы проходов и аналитика)

- **Топология:** **3 шарда × 2 реплики = 6 нод** в primary-регионе, ReplicatedMergeTree.
- **Ключ шардирования (`access_events`):** `cityHash64(object_id)` (равномерное распределение, локальность по объекту).
- **Партиционирование:** `PARTITION BY toYYYYMM(timestamp)` — упрощает архивацию.
- **Order by:** `(object_id, timestamp)` для эффективных диапазонных запросов.
- **TTL:** 36 месяцев горячего хранения, далее `TO VOLUME 'cold_s3'` (S3-backed) либо удаление по политике.
- **DR:** asynchronous replication через `clickhouse-keeper` в DR-регион (lag ≤ 60 с).

### 4.3. Kafka

- **Топология:** **3 broker × 2 AZ** (минимум 3 узла кворума ZooKeeper/KRaft).
- **Конфигурация типового топика:** 30 партиций, replication factor = 3, min.insync.replicas = 2.
- **Ключевые топики:**
  - `passes.events` (created/updated/revoked) — 30 партиций, retention 7 дней.
  - `access.events` — 60 партиций (выше throughput), retention 7 дней (далее всё хранится в ClickHouse).
  - `device.heartbeat` — 30 партиций, compacted-режим (последний heartbeat по `serial`).
  - `device.config` — compacted, retention бессрочно.
  - `oss.events` — 10 партиций, retention 30 дней.
  - `audit.sign` — 30 партиций, retention 30 дней (consumer пишет в `audit_log` ClickHouse).
  - `notifications.send` — 30 партиций, retention 7 дней.
  - `partner.callbacks` — 30 партиций, retention 7 дней.
- **Mirror-maker 2** в DR-регион для топиков `audit.sign` и `oss.events` (юр. значимые).

### 4.4. Redis Cluster

- **Топология:** **3 master × 2 replica = 9 нод** в primary-регионе, slots 16 384.
- **Memory:** 32 ГБ на master, AOF + RDB snapshot.
- **DR:** cold replica в DR-регионе (manual-failover), кэш не критичен для RTO.

### 4.5. Регионы и юрисдикция

- **Primary:** Москва. Облачные провайдеры — **Yandex Cloud** или **VK Cloud** (оба сертифицированы по 152-ФЗ и ФСТЭК-17, дают managed K8s и managed PostgreSQL).
- **DR:** **Санкт-Петербург** или **Екатеринбург** (зависит от availability zones провайдера).
- **Юрисдикция данных:** **только территория Российской Федерации** (152-ФЗ, ст. 18.1; локализация ПДн). S3 RuStack — VK Cloud Storage / Yandex Object Storage / Selectel Cloud Storage.
- **Egress-контроль:** запрет исходящих соединений из VPC к не-российским IP-диапазонам, кроме whitelist (Anthropic API через прокси с RU-фильтрацией, Telegram self-hosted Bot API).

---

## 5. Bottleneck-prevention (компенсирующие меры)

| Узкое место | Природа ограничения | Решение |
|---|---|---|
| **ЕСИА rate-limit** | Публичных лимитов нет, фактические — на согласовании с Минцифры. Превышение приводит к 429 и блокировке IS. | Локальный кэш `userinfo` на 15 мин (см. § 3); согласование увеличенного лимита при подаче на «Commercial IS»; экспоненциальный backoff с jitter; маршрутизация запросов через очередь Kafka `esia.refresh` с конкурентностью ≤ N. |
| **ГИС ЖКХ SOAP latency** | SOAP-вызовы тяжёлые (XML + ЭЦП XAdES-T), latency 0,5–3 с/запрос; нет пакетных endpoints для большинства реестров. | Асинхронные batch-выгрузки через `/NsiService/` (есть нативная поддержка в ГИС ЖКХ); кэш справочников в Redis на 24 ч (см. § 3); фоновые синхронизаторы (Temporal.io workflows) с дневной/недельной частотой. |
| **XAdES-T подпись CPU-bound** | КриптоПро CSP подписывает 10–30 подписей/с/CPU; на 500 RPS финала ОСС или 140 RPS audit это становится узким местом. | HSM-пул КриптоПро HSM 2.0 (отдельный hardware-кластер с пропускной способностью 200+ ops/s); очередь Kafka `audit.sign` с воркерами; батч-подпись по 50–100 событий в одном XAdES-T-блоке (где допустимо юридически). |
| **BLE-валидация на устройстве** | Online-валидация через `device-gateway` дала бы latency 200–500 ms (+ нестабильность сети ПАК). | Локальный кэш до 1 000 активных ключей в EEPROM устройства; синхронизация через `/devices/{serial}/keys/sync` каждые 5 мин или при push-уведомлении; offline-валидация подписи Ed25519 с TTL 24 ч; очередь оффлайн-проходов (до 10 000 записей) с синхронизацией при возврате online. |
| **ClickHouse query overload (отчёты УК)** | Тяжёлые отчёты на горизонт 90 дней могут блокировать insert-поток. | Materialized views для типовых дашбордов (DAU/MAU, проходы по часам); reader-replicas отдельно от writer-узлов; query queue с приоритетами (real-time UI > batch reports). |
| **Kafka consumer lag для notifications** | При burst 500 RPS push в момент финала ОСС consumer может отставать. | Партиционирование по `user_id` (равномерность); горизонтальное масштабирование consumer group (KEDA-autoscaler в K8s по lag-метрике); idempotent producer для гарантии exactly-once на стороне consumer. |
| **PostgreSQL write contention на горячие объекты** | Объект-«миллионник» (5 000+ квартир, 100 000+ пропусков/мес) — потенциальный hotspot. | Sub-sharding внутри объекта по `entrance_id` (эвентуально, при подтверждении проблемы); write-through в Redis для счётчиков. |
| **Anthropic Claude API недоступность / медленные ответы** | Внешний сервис, географическая блокировка возможна. | Локальный fallback на YandexGPT/GigaChat (см. ADR-002); circuit breaker (Hystrix-pattern) с таймаутом 8 с; деградация в "режим FAQ-only" (без LLM, прямые ответы из Qdrant top-K). |

---

## 6. Observability stack

Покрытие трёх слоёв: метрики (Prometheus), логи (Loki), трассы (Tempo) — связаны через OpenTelemetry SDK во всех сервисах.

| Слой | Компонент | Назначение | Развёртывание |
|---|---|---|---|
| Метрики (короткий retention) | **Prometheus** | scrape-метрики со всех сервисов и инфраструктуры (PG-exporter, redis-exporter, kafka-exporter, k8s-state-metrics) | HA-pair в primary-регионе |
| Метрики (долгий retention) | **VictoriaMetrics** | Долгое хранение (1 год+) для capacity planning и долгосрочной операционной отчётности | 3-нодовый кластер |
| Дашборды | **Grafana** | Технические дашборды (по сервису, по слою, по бизнес-метрике) | HA-pair, SSO через ЕСИА для администраторов |
| Логи | **Loki** | Структурированные JSON-логи всех сервисов | 3 ingester + 3 querier; S3-backed storage |
| Трассы | **Tempo** | Distributed tracing запросов через все микросервисы | sampling 1% (head-based) + 100% для error-traces (tail-based) |
| Инструментация | **OpenTelemetry SDK** | Единый стандарт инструментации в Python (`opentelemetry-api`, `opentelemetry-instrumentation-fastapi`) и Go (`go.opentelemetry.io/otel`) | библиотека в каждом сервисе |
| Frontend errors | **Sentry** (self-hosted в РФ) | Ошибки UI, перформанс-метрики Vue/SPA | self-hosted на K8s |
| Backend exceptions | **Sentry** | Все необработанные исключения backend, deprecation warnings | self-hosted на K8s |
| Бизнес-аналитика | **ClickHouse + Superset** (или Metabase) | BI-дашборды для УК, Минцифры, инвесторов | Superset на K8s, ClickHouse-кластер из § 4.2 |
| Алертинг | **Alertmanager** + **PagerDuty** или **OpsGenie** | Маршрутизация и эскалация алертов; интеграция с дежурными расписаниями | Alertmanager HA-pair |
| SIEM (требование ФСТЭК-17) | **MaxPatrol SIEM** | Security events (auth, изменение прав, аудит admin-действий) | по требованиям приказа № 17 |
| Synthetic monitoring | **Blackbox Exporter** + custom k6-сценарии | Проверка работоспособности критических endpoints из внешних точек (РФ-провайдеры) | в каждом регионе РФ-присутствия |

### 6.1. SLI/SLO как единый источник истины

Все SLO определены в YAML-конфиге (Sloth или Pyrra), компилируются в Prometheus alerting rules + Grafana dashboards автоматически. Error budget — 99,9 % uptime → 43,2 мин/мес budget.

---

## 7. Ключевые SLI/SLO и алерты

| Метрика | Что измеряет | Warning | Critical | Реакция |
|---|---|---|---|---|
| `acl_check_p99_latency_ms` | p99 latency ACL-проверки в `acl-svc` | > 100 ms (5 мин окно) | > 300 ms (1 мин окно) | Critical: пользователь физически стоит у двери — немедленный пейджинг дежурного |
| `pass_creation_rate_per_sec` | RPS на `POST /passes` (бизнес-индикатор) | drop > 50 % от baseline (5 мин) | drop > 80 % от baseline (5 мин) | Возможный инцидент с авторизацией / БД |
| `device_offline_count_pct` | % устройств с last_heartbeat > 5 мин | > 5 % | > 10 % | Проблема в `device-gateway` или сетевая |
| `device_offline_count_per_uk` | Кол-во offline ПАК у одной УК | > 30 % устройств УК | > 50 % устройств УК | Эскалация в УК + auto-ticket |
| `esia_userinfo_error_rate` | % HTTP 5xx/timeout от ЕСИА за 5 мин | > 1 % | > 5 % | Возможная блокировка нашей IS, эскалация GR |
| `gis_gkh_sync_lag_hours` | Запаздывание синхронизации справочников ГИС ЖКХ | > 24 ч | > 72 ч | Регуляторное несоответствие |
| `oss_signature_verification_fail_rate` | % голосов ОСС с невалидной XAdES-T-подписью | > 0,1 % | > 1 % | Юридический риск: голосование может быть оспорено |
| `kafka_consumer_lag` | Lag consumer group по любому критическому топику | > 10 000 messages (5 мин) | > 100 000 messages (5 мин) | Authoscaler не справился; ручное масштабирование |
| `postgres_replication_lag_seconds` | Lag репликации PG-Citus master → standby | > 30 с | > 120 с | Угроза RPO; switchover-готовность |
| `clickhouse_insert_lag_seconds` | Лаг очереди inserts в `access_events` | > 30 с | > 300 с | Возможна потеря событий в Kafka retention |
| `xades_t_sign_queue_depth` | Глубина очереди `audit.sign` в Kafka | > 5 000 messages | > 50 000 messages | HSM не справляется, добавить производительности |
| `partner_quota_exceeded_count` | Кол-во отказов партнёрам по превышению квоты за час | > 100 / ч | > 1 000 / ч | Партнёр упёрся в лимит, повод обсудить расширение подписки (а не тех. реакцию) |
| `app_error_rate_5xx` | % HTTP 5xx от общего трафика API | > 0,5 % (5 мин) | > 2 % (5 мин) | Общий health composite-индикатор |
| `cert_expiry_days` | Дни до истечения TLS / mTLS / GOST-сертификатов | < 30 дней | < 7 дней | Заблаговременное обновление сертификатов |
| `disk_free_pct` | % свободного места на дисках PG/ClickHouse/Kafka | < 30 % | < 15 % | Расширение volume / cleanup / архивация |
| `ble_token_offline_hit_rate` | % ACL-проверок, прошедших оффлайн (на устройстве) | < 70 % | < 50 % | Сеть нестабильна, проверить device-gateway |

Алерты маршрутизируются в **PagerDuty** (или OpsGenie) с учётом часа и роли дежурного: критические BLE/ACL — 24/7 пейджинг SRE; ОСС-сигнатуры — рабочее время + on-call ИБ; ГИС ЖКХ-синхронизация — рабочее время GR-команды.

---

## 8. Бизнес-метрики (ClickHouse + Superset/Metabase)

Эти метрики не относятся к health системы, но критичны для управления продуктом и регуляторной отчётности (Минцифры/Минстрой).

> **Примечание для разработчиков.** Конкретные таргеты MRR / LTV / ARPU / CAC / доли B2B-выручки и связанные с ними финансовые витрины выведены в NDA-материалы (`NDA/financial-model.md`). Технически они потребуют тех же источников (ClickHouse + PG `subscription` / `invoice` / `payment` + `partner_key.used`); ниже — продуктовые и операционные KPI без денежных порогов.

| Метрика | Целевое значение к 24 мес | Источник | Витрина |
|---|---|---|---|
| **DAU / MAU по объектам** | DAU 30 % от MAU | `access_events` (ClickHouse) с DISTINCT по `user_id` за окно | Продуктовый дашборд |
| **Доля пропусков, обработанных без участия диспетчера** | ≥ 70 % к Фазе 3 | Соотношение `pass.created.auto` / `pass.created.total` | Продуктовый дашборд |
| **Среднее время реакции УК на заявку** | ≤ 15 мин в Фазе 2, ≤ 5 мин в Фазе 4 | Median(`responded_at` − `created_at`) по тикетам | Дашборд саппорта |
| **Conversion подключения УК** (от лида до hard-launch) | ≥ 40 % | CRM (Bitrix24-зеркалирование) + `uk.contract_signed_at` | Sales-дашборд |
| **Time-to-Live УК** (от заявки до активного объекта) | ≤ 60 дней без ОСС, ≤ 90 с ОСС | Median(`object.onboarded_at` − `lead.created_at`) | Sales-дашборд |
| **Churn УК (помесячно)** | ≤ 2 % | Δ `uk.status='terminated'` помесячно | Sales / CSM-дашборд |
| **Число регионов с активными проектами** | ≥ 10 к Фазе 5 | Distinct `object.region` с active-объектами | GR-дашборд |
| **Доля ПАК с offline-fallback в текущем месяце** | < 1 % проходов | `access_events.method = 'ble_offline'` | SRE-дашборд (но и продуктовый сигнал) |
| **NPS жителей** | ≥ 40 в Фазе 2 | CSAT-сурвей через push в Госуслуги.Дом | Продуктовый дашборд |

---

## 9. Стратегия тестирования

### 9.1. REUSE из pass24-servicedesk

- **pytest** + **pytest-asyncio** + **httpx-async** — основа для unit и integration backend.
- **Playwright** — E2E-тесты UI (Vue 3 + PrimeVue), сценарии «Житель заказывает пропуск», «УК создаёт ОСС», «Диспетчер обрабатывает заявку».
- **Coverage target:** **≥ 80 %** на критических модулях (`acl-svc`, `passes-svc`, `oss-svc`, `partners-svc`, `billing-svc`).
- **238 тестовых кейсов** из pass24-servicedesk используются как стартовая база для модулей `tickets`, `knowledge`, `notifications`, `assistant`, `customers`, `auth` (после форка).
- **CI:** GitHub Actions workflow `ops-run-tests.yml` (REUSE) — параллельный запуск тестов, fail-fast.

### 9.2. UPGRADE / NEW

#### 9.2.1. Contract testing

- **Pact** или **Schemathesis** для всех публичных API:
  - **Mobile BFF** (Госуслуги.Дом) — провайдер-контракт согласовывается с ПАО МРТС.
  - **B2B Gateway** — контракты Yandex GO / OZON / WB (издаёт ESD как provider, потребляют партнёры).
  - **OEM Gateway «Госуслуги.ДОМ Ready»** — контракты с производителями домофонов.
- Контракты хранятся в Pact Broker (self-hosted в РФ); каждый release blocked если contract verification fail.

#### 9.2.2. Нагрузочное тестирование

Целевые сценарии (на staging-кластере, эквивалентном prod по топологии, с production-like данными):

| Сценарий | Tool | Профиль | Acceptance criteria |
|---|---|---|---|
| Утренний пик ACL | **k6** | 1 400 RPS sustained 30 мин на `POST /v1/access/check` | p99 < 100 ms, error rate < 0,1 % |
| Вечерняя доставка партнёров | **Gatling** | ramp 0 → 3 000 RPS за 5 мин на `POST /b2b/v1/keys/issue`, hold 30 мин | p99 < 200 ms, error rate < 0,5 % |
| Финал ОСС-голосования | **Locust** | 500 RPS на `POST /oss/{id}/vote` с XAdES-T payload | p95 < 600 ms, signature_verify_fail = 0 |
| Сбор пропусков жителем (UI) | **k6 (browser-mode)** | 200 concurrent users, click-through сценарии | p95 page load < 2 с |
| ПАК heartbeat | **k6** | 5 000 виртуальных устройств, 1 publish / 30 с | устойчивая обработка 170 RPS, lag NATS < 50 ms |
| Holistic (соединение всех потоков) | **Gatling** | Mix 60 % ACL + 20 % partners + 10 % UI + 10 % heartbeat | composite SLO выполняются все одновременно |

Запуски:
- **Pre-release:** перед каждым major-релизом (раз в спринт).
- **Smoke load:** ежедневно ночью, baseline 10 % от продакшен-нагрузки.
- **Capacity tests:** раз в квартал — поиск точки перегиба.

#### 9.2.3. Chaos Engineering

- **Chaos Mesh** (или Litmus) в K8s-кластере staging:
  - Kill-pod random-сценарий для всех сервисов.
  - Network partition между AZ.
  - Резкое падение PostgreSQL primary → проверка switchover.
  - 30 % CPU stress на ноды HSM-кластера.
  - DNS-отказ ГИС ЖКХ / ЕСИА → проверка graceful degradation.
- **Game Days** раз в квартал (организует SRE + ИБ): полная имитация инцидента (потеря AZ, утечка ключа, DDoS).

#### 9.2.4. Security testing

- **OWASP ZAP** (DAST) — полное сканирование staging-API раз в спринт.
- **Trivy** — сканирование Docker-образов на CVE в pipeline (block-релиз при HIGH+).
- **Snyk** — сканирование зависимостей Python (`requirements.txt`) и npm (`package.json`).
- **Bandit** (Python) и **gosec** (Go) — статический анализ.
- **Ручной penetration test** — внешним лицензиатом ФСТЭК один раз перед подачей на аттестацию (Фаза 2) и далее ежегодно.
- **Bug bounty** — запуск с Фазы 3 на платформе с РФ-юрисдикцией (например, Standoff 365).

#### 9.2.5. Compliance testing

- **Автоматическая проверка локализации ПДн в РФ:**
  - Healthcheck `/admin/compliance/data-residency` обходит все хранилища (PG-Citus, ClickHouse, Redis, S3-бакеты, Kafka-брокеры) и резолвит публичные IP в ASN/geo. Любое не-RU попадание → critical-алерт.
  - Включается в ежедневный smoke-suite в CI.
- **Автоматическая проверка истечения сертификатов** (см. § 7, метрика `cert_expiry_days`).
- **Проверка консистентности audit-цепочки:** ежедневно проверяется, что в `audit_log` (ClickHouse) нет «дыр» по `event_id` и каждая запись имеет валидную XAdES-T-подпись.
- **Проверка целостности журнала ОСС-голосов:** ежедневно по каждому активному ОСС проверяется hash-цепочка голосов и валидность ЕСИА-подписей.

---

## 10. Связанные документы

- [architecture.md](./architecture.md) — компонентная и инфраструктурная архитектура, обоснование выбора стека.
- [data-model.md](./data-model.md) — физическая модель данных, связи сущностей, ключи шардинга.
- [adr.md](./adr.md) — ADR-002 (стек), ADR-005 (immutable audit XAdES-T), ADR-006 (RBAC).
- [regulatory-roadmap.md](./regulatory-roadmap.md) — регуляторные требования (152-ФЗ, ФСТЭК-17), которые формируют compliance-блок NFR.
- [risks-register.md](./risks-register.md) — реестр рисков, в том числе сценарии деградации (отказ ЕСИА, отказ ГИС ЖКХ, потеря AZ).
- [operations-playbooks.md](./operations-playbooks.md) — операционные регламенты реагирования на алерты из § 7.
- [api-map.md](./api-map.md) — список endpoints, на которые завязаны нагрузочные сценарии.
