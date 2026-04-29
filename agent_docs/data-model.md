# Модель данных ESD ServiceDesk

> Описание основных сущностей платформы Оператора ЕСД с указанием стратегии шардинга, партиционирования, индексов и архивации. Формат — псевдо-Python (SQLModel) для транзакционных таблиц PostgreSQL и SQL DDL для аналитического слоя ClickHouse и метрик TimescaleDB. Подробное ТЗ модулей — в `functional-spec.md`. API-эндпоинты, оперирующие сущностями — в `api-map.md`. Решения по технологиям — ADR-002 в `adr.md`.

---

## 1. Сущности PostgreSQL (OLTP)

> Шардинг — Citus PostgreSQL, шард-ключ `object_id` для горячих таблиц. Реплики чтения — асинхронные стримы в другую AZ. Все таблицы имеют `created_at`, `updated_at` (заполняются триггером).

### 1.1. Identity и пользователи

```python
class User(SQLModel, table=True):
    id: UUID = Field(primary_key=True)
    esia_id: str | None = Field(unique=True, index=True)  # OID из ЕСИА
    snils_hash: str | None = Field(index=True)  # для матчинга жителей
    email: str | None = Field(index=True)
    full_name: str
    phone: str | None
    role: UserRole  # enum, см. ADR-006 (16 ролей)
    is_active: bool = True
    uk_id: UUID | None = Field(foreign_key="uk.id")  # для PROPERTY_MANAGER, DISPATCHER, SECURITY_GUARD
    installer_id: UUID | None = Field(foreign_key="installer.id")
    last_login_at: datetime | None
    created_at: datetime
    updated_at: datetime

class Session(SQLModel, table=True):  # хранится в Redis Cluster, не в PG
    id: UUID
    user_id: UUID
    jwt_jti: str  # для отзыва
    expires_at: datetime
    ip_address: str
    user_agent: str

class M2MClient(SQLModel, table=True):
    id: UUID = Field(primary_key=True)
    client_id: str = Field(unique=True, index=True)
    client_secret_hash: str
    partner_id: UUID | None = Field(foreign_key="partner.id")
    api_tier: Literal["read", "rw_webhooks", "full"]
    mtls_cert_fingerprint: str
    is_active: bool
    created_at: datetime

class DeviceCredential(SQLModel, table=True):
    id: UUID = Field(primary_key=True)
    device_id: UUID = Field(foreign_key="device.id", index=True)
    cert_fingerprint: str = Field(unique=True)
    public_key: str
    valid_from: datetime
    valid_until: datetime  # rotation каждые 30 дней
    is_active: bool
```

### 1.2. УК и юр. контур

```python
class UK(SQLModel, table=True):
    id: UUID = Field(primary_key=True)
    inn: str = Field(unique=True, index=True)
    ogrn: str = Field(index=True)
    name: str
    license_number: str  # лицензия 191-ФЗ
    license_valid_until: date
    gis_gkh_guid: str | None  # GUID в реестре УК ГИС ЖКХ
    status: Literal["prospect", "active", "suspended", "terminated"]
    contract_signed_at: datetime | None
    subscription_tier: Literal["basic", "standard", "professional", "multi_object"]
    health_score: float  # 0-100, считается из ClickHouse
    csm_user_id: UUID | None  # ответственный CSM
    created_at: datetime
    updated_at: datetime

class UKContract(SQLModel, table=True):
    id: UUID
    uk_id: UUID = Field(foreign_key="uk.id", index=True)
    contract_number: str
    signed_at: datetime
    valid_until: datetime | None
    pdf_storage_path: str  # S3
    xades_t_signature: str  # подписан Оператором ЕСД
```

### 1.3. Объекты и пространственная модель

```python
class Object(SQLModel, table=True):
    """SHARD KEY: id. Шардируется Citus."""
    id: UUID = Field(primary_key=True)
    uk_id: UUID = Field(foreign_key="uk.id", index=True)
    fias_guid: str | None = Field(index=True)  # связь с ФИАС/ГАР
    address: str
    region: str
    octmo: str = Field(index=True)
    object_type: Literal["mkd", "village", "office", "kp"]
    geo: Point  # PostGIS geography(Point, 4326)
    status: Literal["pending_oss", "active", "suspended"]
    oss_decision_id: UUID | None
    onboarded_at: datetime | None
    created_at: datetime
    updated_at: datetime

class Premise(SQLModel, table=True):
    """SHARD KEY: object_id."""
    id: UUID
    object_id: UUID = Field(foreign_key="object.id", index=True)
    apartment_number: str
    floor: int
    entrance: int
    area_sqm: float  # для расчёта кворума ОСС
    is_residential: bool
    fias_premise_guid: str | None

class AccessPoint(SQLModel, table=True):
    """SHARD KEY: object_id."""
    id: UUID
    object_id: UUID = Field(foreign_key="object.id", index=True)
    name: str  # "Подъезд 1", "Шлагбаум на въезде"
    type: Literal["entrance", "barrier", "gate", "parking", "wicket"]
    device_id: UUID | None = Field(foreign_key="device.id")
    is_active: bool
```

### 1.4. Жители и согласия

```python
class Resident(SQLModel, table=True):
    """SHARD KEY: object_id."""
    id: UUID
    user_id: UUID | None = Field(foreign_key="user.id")  # после link-esia
    object_id: UUID = Field(foreign_key="object.id", index=True)
    premise_id: UUID = Field(foreign_key="premise.id")
    role: Literal["owner", "tenant", "co_owner"]
    ownership_share: Decimal  # доля для расчёта голосов ОСС
    verified_via_esia: bool = False
    consented_at: datetime  # согласие на обработку ПДн через договор УК
    deleted_at: datetime | None  # soft-delete, архив 7 лет

class ResidentConsent(SQLModel, table=True):
    id: UUID
    resident_id: UUID
    contract_version: str  # связь с версией договора управления
    consented_at: datetime
    consent_text_hash: str  # SHA-256 hash текста согласия
```

### 1.5. Пропуска (ядро)

```python
class Pass(SQLModel, table=True):
    """SHARD KEY: object_id. Партиционирование by month."""
    id: UUID
    object_id: UUID = Field(foreign_key="object.id", index=True)
    issued_by_user_id: UUID  # житель / УК / партнёр (через partner_id)
    pass_type: Literal["one_time", "temporary", "permanent", "courier_through"]
    status: Literal["draft", "active", "used", "expired", "revoked", "rejected"]
    guest_full_name: str | None
    guest_phone: str | None
    guest_car_plate: str | None  # для шлагбаума
    valid_from: datetime
    valid_until: datetime
    access_point_ids: list[UUID]  # массив для courier_through
    qr_token: str  # uuid4, индекс уникальный
    ble_token_hash: str  # SHA-256 от BLE-токена
    partner_id: UUID | None = Field(foreign_key="partner.id")
    partner_order_id: str | None  # external ID партнёра
    requires_uk_approval: bool = False
    approved_by_user_id: UUID | None
    approved_at: datetime | None
    used_at: datetime | None
    revoked_at: datetime | None
    revoked_reason: str | None
    created_at: datetime
    updated_at: datetime

class Key(SQLModel, table=True):
    """Постоянный ключ жителя (привязка к мастер-ключу). SHARD KEY: object_id."""
    id: UUID
    resident_id: UUID = Field(foreign_key="resident.id")
    object_id: UUID = Field(foreign_key="object.id", index=True)
    master_key_id: UUID | None  # привязка к мастеру (для group revoke)
    key_type: Literal["ble", "nfc", "qr_static"]
    key_data_encrypted: str  # зашифрованный материал
    valid_until: datetime | None
    is_active: bool
    issued_at: datetime
    revoked_at: datetime | None
```

### 1.6. ОСС-конструктор

```python
class OSSTemplate(SQLModel, table=True):
    id: UUID
    code: str = Field(unique=True)  # "connect_to_esd", "install_pak"
    title: str
    description_md: str
    questions: JSON  # массив вопросов
    quorum_required_pct: float  # обычно 50% или 2/3
    voting_duration_days: int = 10
    is_active: bool

class OSS(SQLModel, table=True):
    """SHARD KEY: object_id."""
    id: UUID
    object_id: UUID = Field(foreign_key="object.id", index=True)
    initiator_user_id: UUID
    template_id: UUID = Field(foreign_key="osstemplate.id")
    title: str
    description_md: str
    starts_at: datetime
    ends_at: datetime
    status: Literal["draft", "active", "completed", "failed", "cancelled"]
    quorum_required_pct: float
    quorum_pct: Decimal | None  # фактический
    temporal_workflow_id: str  # ID workflow в Temporal.io
    protocol_pdf_path: str | None  # S3 путь после завершения
    protocol_xades_t_signature: str | None

class OSSQuestion(SQLModel, table=True):
    id: UUID
    oss_id: UUID = Field(foreign_key="oss.id", index=True)
    question_number: int
    question_text: str
    options: JSON  # ["for", "against", "abstain"] или специфичные

class OSSVote(SQLModel, table=True):
    """SHARD KEY: object_id."""
    id: UUID
    oss_id: UUID = Field(foreign_key="oss.id", index=True)
    question_id: UUID
    resident_id: UUID
    premise_id: UUID
    decision: Literal["for", "against", "abstain"]
    voted_at: datetime
    esia_signature_pkcs7: str  # PKCS#7 detached, GOST 2012
    signature_verified: bool = False
    verification_error: str | None
```

### 1.7. Биллинг

```python
class Subscription(SQLModel, table=True):
    id: UUID
    uk_id: UUID = Field(foreign_key="uk.id", index=True)
    object_id: UUID = Field(foreign_key="object.id")
    tier: Literal["basic", "standard", "professional", "multi_object"]
    monthly_fee_rub: int
    started_at: datetime
    next_billing_at: datetime
    status: Literal["active", "grace", "suspended", "terminated"]
    objects_quota: int
    passes_quota_per_month: int
    sms_quota_per_month: int
    api_tier: Literal["read", "rw_webhooks", "full"]
    auto_upgrade_enabled: bool = False

class UsageMetric(SQLModel, table=True):
    """Раз в час агрегаты, источник — ClickHouse."""
    id: UUID
    subscription_id: UUID
    period_start: datetime
    period_end: datetime
    passes_used: int
    sms_used: int
    api_calls: int
    overage_rub: Decimal

class Invoice(SQLModel, table=True):
    id: UUID
    uk_id: UUID
    period_start: datetime
    period_end: datetime
    subtotal_rub: Decimal
    overage_rub: Decimal
    total_rub: Decimal
    status: Literal["draft", "issued", "paid", "overdue", "cancelled"]
    diadoc_doc_id: str | None  # ID в Контур.Диадок
    paid_at: datetime | None

class Payment(SQLModel, table=True):
    id: UUID
    invoice_id: UUID
    provider: Literal["yandex_kassa", "tinkoff_kassa", "manual"]
    provider_payment_id: str
    amount_rub: Decimal
    paid_at: datetime
```

### 1.8. Партнёры и B2B

```python
class Partner(SQLModel, table=True):
    id: UUID
    name: str  # "Yandex GO", "OZON", "WildBerries"
    inn: str = Field(unique=True)
    api_tier: Literal["read", "rw_webhooks", "full"]
    monthly_quota_keys: int
    used_keys_current_month: int
    is_active: bool
    contract_signed_at: datetime
    csm_user_id: UUID

class PartnerKey(SQLModel, table=True):
    """Журнал выданных партнёрских ключей. SHARD KEY: partner_id."""
    id: UUID
    partner_id: UUID = Field(foreign_key="partner.id", index=True)
    pass_id: UUID = Field(foreign_key="pass.id")
    partner_order_id: str = Field(index=True)  # external ID
    issued_at: datetime
    used_at: datetime | None
    callback_sent_at: datetime | None
    callback_attempts: int = 0

class PartnerWebhookSubscription(SQLModel, table=True):
    id: UUID
    partner_id: UUID
    webhook_url: str
    events: list[str]  # ["pass.issued", "pass.used", ...]
    secret: str  # для HMAC-подписи
    is_active: bool
```

### 1.9. ПАК-устройства

```python
class Device(SQLModel, table=True):
    id: UUID
    serial_number: str = Field(unique=True, index=True)
    model: str
    manufacturer: str  # "BAS-IP", "SLINEX", "Beward"
    firmware_version: str
    object_id: UUID | None = Field(foreign_key="object.id")
    access_point_id: UUID | None = Field(foreign_key="accesspoint.id")
    status: Literal["registered", "provisioned", "online", "offline", "decommissioned"]
    last_heartbeat: datetime | None
    public_key: str  # ECC P-256 или GOST для криптообмена
    installer_job_id: UUID | None
    provisioned_at: datetime | None

class DeviceFirmware(SQLModel, table=True):
    id: UUID
    version: str
    model: str  # для какой модели ПАК
    storage_path: str  # S3 путь к подписанному образу
    signature_xades_t: str
    released_at: datetime
    is_active: bool

class DeviceConfig(SQLModel, table=True):
    id: UUID
    device_id: UUID = Field(foreign_key="device.id")
    config_json: JSON
    applied_at: datetime | None
```

### 1.10. Биржа монтажников и LMS

```python
class Installer(SQLModel, table=True):
    id: UUID
    user_id: UUID = Field(foreign_key="user.id")
    company_name: str | None
    region: str
    geo: Point
    rating: float = 0.0  # 0-5
    completed_jobs: int = 0
    is_active: bool

class InstallerCertification(SQLModel, table=True):
    id: UUID
    installer_id: UUID
    course_id: UUID
    level: Literal["lite", "standard", "pro", "expert"]
    issued_at: datetime
    valid_until: date
    xades_t_signature: str  # сертификат с подписью Оператора

class InstallerJob(SQLModel, table=True):
    """SHARD KEY: object_id."""
    id: UUID
    object_id: UUID = Field(index=True)
    uk_id: UUID
    installer_id: UUID | None
    status: Literal["open", "claimed", "in_progress", "tested", "completed", "rejected"]
    devices_to_install: int
    price_rub: int
    commission_rub: int
    scheduled_at: datetime
    completed_at: datetime | None
    acceptance_test_passed: bool
    acceptance_test_log: JSON
    photos_storage_paths: list[str]

class Course(SQLModel, table=True):
    id: UUID
    code: str
    title: str
    target_level: Literal["lite", "standard", "pro", "expert"]
    pass_threshold_pct: int = 80

class Lesson(SQLModel, table=True):
    id: UUID
    course_id: UUID
    sequence: int
    title: str
    content_md: str
    video_url: str | None
    duration_min: int

class ExamAttempt(SQLModel, table=True):
    id: UUID
    user_id: UUID
    course_id: UUID
    started_at: datetime
    completed_at: datetime | None
    score_pct: int | None
    passed: bool | None
```

### 1.11. Service Desk

```python
class Ticket(SQLModel, table=True):
    """REUSE из pass24-servicedesk. SHARD KEY: uk_id."""
    id: UUID
    uk_id: UUID | None
    object_id: UUID | None
    device_id: UUID | None
    installer_id: UUID | None
    title: str
    description: str
    status: Literal["new", "in_progress", "waiting_client", "resolved", "closed"]
    priority: Literal["low", "medium", "high", "critical"]
    assigned_to_user_id: UUID | None
    sla_due_at: datetime
    resolved_at: datetime | None
    csat_score: int | None  # 1-5
    csat_comment: str | None

class Comment(SQLModel, table=True):
    id: UUID
    ticket_id: UUID = Field(index=True)
    author_user_id: UUID
    body_md: str
    is_internal: bool = False
    created_at: datetime

class Attachment(SQLModel, table=True):
    id: UUID
    ticket_id: UUID
    storage_path: str  # S3
    filename: str
    size_bytes: int
    mime_type: str
```

### 1.12. Knowledge & LMS

```python
class Article(SQLModel, table=True):
    id: UUID
    slug: str = Field(unique=True, index=True)
    title: str
    body_md: str
    category: str
    tags: list[str]
    is_public: bool
    views_count: int = 0
    likes_count: int = 0
    embedding_id: str | None  # связь с Qdrant
```

### 1.13. Notifications

```python
class Notification(SQLModel, table=True):
    """SHARD KEY: user_id. Партиционирование by month, TTL 6 мес."""
    id: UUID
    user_id: UUID = Field(index=True)
    channel: Literal["push", "email", "sms", "telegram", "whatsapp"]
    template_code: str
    payload: JSON
    status: Literal["queued", "sent", "delivered", "failed"]
    error: str | None
    sent_at: datetime | None
```

---

## 2. ClickHouse OLAP-таблицы

> Append-only, partitioned by month, replicated 2× в каждой из 3 шардов. TTL 36 мес → S3 cold storage.

### 2.1. Журнал проходов (`access_events`)

```sql
CREATE TABLE access_events ON CLUSTER esd_cluster
(
    event_id UUID,
    timestamp DateTime64(3, 'Europe/Moscow'),
    device_id UUID,
    access_point_id UUID,
    object_id UUID,
    pass_id Nullable(UUID),
    user_id Nullable(UUID),
    method LowCardinality(String),         -- 'ble', 'qr', 'nfc', 'facial', 'lpr', 'manual'
    result LowCardinality(String),         -- 'granted', 'denied'
    deny_reason Nullable(String),
    partner_id Nullable(UUID),
    raw_signed_payload String,             -- XAdES-T blob, audit-trail
    ingested_at DateTime DEFAULT now()
)
ENGINE = ReplicatedMergeTree('/clickhouse/access_events/{shard}', '{replica}')
PARTITION BY toYYYYMM(timestamp)
ORDER BY (object_id, timestamp, event_id)
TTL timestamp + INTERVAL 36 MONTH TO VOLUME 'cold_s3';

CREATE MATERIALIZED VIEW access_events_hourly_mv ON CLUSTER esd_cluster
ENGINE = SummingMergeTree
PARTITION BY toYYYYMM(hour)
ORDER BY (object_id, hour, method, result)
AS SELECT
    object_id,
    toStartOfHour(timestamp) AS hour,
    method,
    result,
    count() AS events_count
FROM access_events
GROUP BY object_id, hour, method, result;
```

### 2.2. Метрики устройств (`device_metrics_oltp`)

> Альтернатива — TimescaleDB hypertable. Выбор зависит от объёма (5000 устройств × 1/30s = ~14 М точек/сутки).

```sql
CREATE TABLE device_metrics ON CLUSTER esd_cluster
(
    device_id UUID,
    timestamp DateTime64(3),
    uptime_seconds UInt32,
    temperature_c Float32,
    signal_strength_dbm Int8,
    cpu_pct Float32,
    memory_pct Float32,
    last_access_event_id Nullable(UUID),
    firmware_version LowCardinality(String)
)
ENGINE = ReplicatedMergeTree('/clickhouse/device_metrics/{shard}', '{replica}')
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (device_id, timestamp)
TTL timestamp + INTERVAL 6 MONTH;
```

### 2.3. Audit log (`audit_log`)

```sql
CREATE TABLE audit_log ON CLUSTER esd_cluster
(
    event_id UUID,
    timestamp DateTime64(3),
    tenant_id UUID,                        -- UK или партнёр
    actor_user_id Nullable(UUID),
    actor_type LowCardinality(String),     -- 'user', 'partner', 'system', 'device'
    action LowCardinality(String),         -- 'pass.issued', 'oss.voted', 'device.firmware.updated'
    target_entity LowCardinality(String),  -- 'Pass', 'OSS', 'Device'
    target_id UUID,
    payload String,                        -- JSON snapshot
    xades_t_signature String,              -- подпись Оператора
    ip_address String,
    user_agent String
)
ENGINE = ReplicatedMergeTree('/clickhouse/audit_log/{shard}', '{replica}')
PARTITION BY toYYYYMM(timestamp)
ORDER BY (tenant_id, timestamp, event_id)
SETTINGS index_granularity = 8192;
-- Без TTL: бессрочное хранение для регуляторного аудита.
```

---

## 3. Стратегия шардинга, партиционирования и архивации

| Таблица | Шардинг | Партиционирование | Архивация / TTL |
|---|---|---|---|
| `pass` | Citus by `object_id` | by month | через 12 мес → S3 (XAdES-T-подписанные дампы) |
| `access_event` (CH) | ClickHouse 3 шарда | by month | TTL 36 мес → S3 cold |
| `oss`, `oss_vote` | Citus by `object_id` | — | бессрочно (юр. документ) |
| `device_metrics` (CH) | ClickHouse 3 шарда | by day | TTL 6 мес |
| `audit_log` (CH) | ClickHouse by `tenant_id` | by month | бессрочно с XAdES-T |
| `notification` | Citus by `user_id` | by month | TTL 6 мес |
| `resident` | Citus by `object_id` | — | soft-delete + архив 7 лет (152-ФЗ) |
| `object`, `premise`, `accesspoint` | Citus by `object_id` (для object — by `id`) | — | — |
| `uk` | без шардинга (мало записей) | — | — |
| `device` | Citus by `object_id` | — | — |
| `installer`, `installer_job` | Citus by `object_id` (job), без шардинга (installer) | — | — |
| `subscription`, `invoice`, `payment` | Citus by `uk_id` | — | бессрочно (бухгалтерский учёт) |
| `partner_key` | Citus by `partner_id` | by month | TTL 36 мес |
| `ticket`, `comment`, `attachment` | Citus by `uk_id` | — | архив через 24 мес → cold |
| `article` | без шардинга | — | — |

---

## 4. Связи и индексы (упрощённая ER-диаграмма)

```
                                    ┌──────────┐
                                    │   USER   │
                                    └────┬─────┘
                       ┌─────────────────┼─────────────────┐
                       ▼                 ▼                 ▼
                 ┌──────────┐     ┌──────────┐      ┌──────────┐
                 │ RESIDENT │     │   UK     │      │INSTALLER │
                 └────┬─────┘     └────┬─────┘      └────┬─────┘
                      │                │                 │
                      │           ┌────▼─────┐           │
                      │           │ CONTRACT │           │
                      │           └──────────┘           │
                      │                │                 │
                      │           ┌────▼─────┐           │
                      └──────────►│  OBJECT  │◄──────────┘
                                  └────┬─────┘
                  ┌──────────────┬─────┼──────────────┬──────────────┐
                  ▼              ▼     ▼              ▼              ▼
            ┌──────────┐  ┌──────────┐ ┌─────────┐ ┌────────┐ ┌──────────┐
            │ PREMISE  │  │ACCESS_PT │ │  DEVICE │ │  PASS  │ │   OSS    │
            └────┬─────┘  └────┬─────┘ └────┬────┘ └────┬───┘ └────┬─────┘
                 │             │            │           │          │
                 │       ┌─────┘            │           │          │
                 │       │                  │           │     ┌────▼─────┐
                 ▼       ▼                  ▼           ▼     │ OSS_VOTE │
              [связь со списком access_point_ids в Pass]      └──────────┘
                                                  │
                                            ┌─────▼──────┐
                                            │ACCESS_EVENT│
                                            │ (ClickHouse)│
                                            └─────────────┘

  ┌──────────┐         ┌────────────────┐         ┌──────────┐
  │ PARTNER  │────────►│  PARTNER_KEY   │────────►│  PASS    │
  └──────────┘         └────────────────┘         └──────────┘

  ┌──────────────┐  ┌────────────┐  ┌────────────┐
  │ SUBSCRIPTION │──│   USAGE    │──│  INVOICE   │
  └──────────────┘  └────────────┘  └────────────┘

  ┌──────────┐  ┌──────────────┐  ┌────────────────────┐
  │ INSTALLER│──│ INSTALLER_JOB│──│  ACCEPTANCE_TEST   │
  └──────────┘  └──────────────┘  └────────────────────┘

  ┌──────────┐  ┌────────────┐  ┌──────────┐
  │  TICKET  │──│  COMMENT   │──│ATTACHMENT│
  └──────────┘  └────────────┘  └──────────┘
```

### Ключевые уникальные индексы

- `User.esia_id` — уникальный, частичный (где не NULL).
- `User.email` — уникальный, частичный.
- `Pass.qr_token` — уникальный.
- `UK.inn` — уникальный.
- `Device.serial_number` — уникальный.
- `Object.fias_guid` — частичный, уникальный.
- `M2MClient.client_id` — уникальный.
- `OSSTemplate.code` — уникальный.

### Композитные индексы (для горячих путей)

- `(Pass.object_id, Pass.status, Pass.valid_until)` — для acl-svc lookup.
- `(AccessEvent.object_id, timestamp)` — основной ORDER BY ClickHouse.
- `(Resident.object_id, premise_id)` — для расчёта кворума ОСС.
- `(InstallerJob.object_id, status)` — для биржи.

### GIN-индексы

- `Pass.access_point_ids` — для обратного поиска «какие пропуска относятся к точке доступа».
- `Article.tags` — для поиска статей по тегам.
- `Object.geo` — GIST-индекс PostGIS для поиска объекта по координатам (B2B `/objects/lookup`).

---

## Связанные документы

- `architecture.md` — технологический стек (Citus, ClickHouse, Qdrant, TimescaleDB, Redis Cluster, Kafka).
- `functional-spec.md` — функциональное ТЗ модулей, использующих эти сущности.
- `api-map.md` — API-эндпоинты, оперирующие сущностями.
- `load-profile.md` — нагрузочный профиль на хранилища (RPS, IOPS, объёмы).
- `adr.md` — ADR-002 (стек), ADR-005 (XAdES-T audit log).
- `regulatory-roadmap.md` — требования по 152-ФЗ к шифрованию ПДн и срокам хранения.
