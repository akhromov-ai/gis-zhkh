# Функциональное ТЗ модулей ESD ServiceDesk

> Детальное функциональное ТЗ к подсистемам платформы Оператора ЕСД. Документ структурирован по микросервисам (bounded contexts). Для каждого модуля приведены: назначение, ключевые user stories по ролям, API-эндпоинты (детальная карта — в `api-map.md`), основные сущности данных (полная модель — в `data-model.md`) и пометки REUSE / UPGRADE / NEW относительно базового стека `pass24-servicedesk`.
>
> Контекст продукта — в `AGENTS.md`. Архитектурные слои и инфраструктура — в `architecture.md`. Архитектурные решения — в `adr.md` (особенно ADR-002 про стек, ADR-003 про тарифы, ADR-004 про Госуслуги.Дом-only стратегию, ADR-005 про XAdES-T audit, ADR-006 про RBAC).

---

## Условные обозначения

- **REUSE** — переиспользуем 1-в-1 паттерн/код из `pass24-servicedesk`.
- **UPGRADE** — берём паттерн из `pass24-servicedesk`, но усиливаем под high-load и госконтур.
- **NEW** — добавляем впервые специально под предметку ЕСД.

---

## Модуль identity-svc — Авторизация и аутентификация

### Назначение

Единая точка идентификации и аутентификации жителей, сотрудников УК, диспетчеров, охраны, монтажников, B2B-партнёров и устройств. Главный канал для конечного пользователя — ЕСИА (OAuth 2.0 / OpenID Connect, GOST Р 34.10-2012). Внутренние сервисы общаются по JWT RS256 с ротацией ключей через Vault. Партнёрские системы — по Client Credentials grant + mTLS. ПАК-устройства — по mTLS-сертификатам с криптопроцессоров.

### User stories

- **Житель.** Открываю Госуслуги.Дом → раздел «Заказать пропуск» → SSO-сессия ЕСИА переиспользуется без повторного ввода логина-пароля → попадаю в кабинет ESD.
- **УК-менеджер.** Захожу в веб-кабинет УК через ЕСИА с ЭЦП юр. лица (КриптоПро EDS Browser Plug-in) → получаю токен с ролью `PROPERTY_MANAGER` и `uk_id` в claims.
- **Диспетчер.** Захожу в веб-кабинет → получаю роль `DISPATCHER` с ограничением `object_ids` по моей смене.
- **Охранник.** Локальный вход на стационарном терминале объекта по ЕСИА + биометрия → роль `SECURITY_GUARD`, ограничение по конкретному `access_point_id`.
- **B2B-партнёр (Yandex GO/OZON/WB).** Backend-сервис партнёра запрашивает M2M-токен по `client_id`/`client_secret` (Client Credentials) + клиентский TLS-сертификат → роль `PARTNER_API` с лимитами по `Partner.api_tier`.
- **ПАК-устройство.** При первой загрузке выполняет provisioning-flow по предустановленному device-сертификату → получает mTLS-канал и долгоживущий device-token.
- **Сотрудник Оператора ЕСД.** Логин через корпоративный SSO (Keycloak) с ролями `SUPPORT_AGENT_L1` / `SUPPORT_AGENT_L2` / `ADMIN` / `CISO` / `COMPLIANCE_AUDITOR`.

### API (краткий перечень)

- `POST /auth/esia/start` — инициация OIDC-flow с ЕСИА.
- `POST /auth/esia/callback` — обмен `code` на токены, обогащение профиля.
- `GET /auth/esia/userinfo` — обращение к userinfo-endpoint ЕСИА (с локальным кэшем 15 мин).
- `POST /auth/refresh` — обновление JWT.
- `POST /auth/logout` — отзыв сессии.
- `POST /auth/sign-document` — подписать документ ЭЦП с КриптоПро CSP.
- `POST /auth/partner-token` — Client Credentials grant для B2B.
- `POST /auth/device-provision` — initial provisioning ПАК-устройства.
- `GET /auth/me` — профиль текущего пользователя.
- `POST /auth/esia/link-uk` — привязка УК-менеджера к юр. лицу через ЕСИА.

Полная карта — см. `api-map.md`.

### Ключевые сущности

`User`, `Session`, `EsiaProfile`, `Role`, `Permission`, `ApiKey`, `M2MClient`, `DeviceCredential`. Полная модель — `data-model.md`.

### REUSE / UPGRADE / NEW

- **REUSE.** Структура `backend/auth/` из `pass24-servicedesk`: FastAPI dependency `get_current_user`, JWT middleware, RBAC-декораторы, password reset flow для внутренних сотрудников.
- **UPGRADE.** JWT — переход с HS256 на RS256 с ротацией ключей через HashiCorp Vault. Расширение ролей с 4 (RESIDENT/PROPERTY_MANAGER/SUPPORT_AGENT/ADMIN) до 16 (см. ADR-006). Хранение sessions в Redis Cluster (вместо PostgreSQL).
- **NEW.** ЕСИА OIDC-flow, PKCS#7 detached signature по GOST Р 34.10-2012, интеграция КриптоПро CSP 5.0 + КриптоПро HSM 2.0 + КриптоПро EDS Browser Plug-in для клиентских ЭЦП. Self-signed сертификаты не поддерживаются (запрет с 01.04.2020). M2M Client Credentials grant для партнёров. Device provisioning flow для ПАК.

---

## Модуль object-svc — Объекты и точки доступа

### Назначение

Реестр объектов недвижимости (МКД, коттеджные посёлки, офисные центры, бизнес-парки) и принадлежащих им точек доступа (подъезды, шлагбаумы, калитки, парковки). Источник адресной нормализации — ФИАС / ГАР через `fias.nalog.ru`. Для МКД дополнительная синхронизация с реестрами ГИС ЖКХ через SOAP API `open-gkh.ru`.

### User stories

- **УК-админ.** Добавляю новый объект, нажимаю «найти в ГИС ЖКХ по адресу» → система ищет МКД в `PremisesBase` → подтягивает `fias_guid`, `octmo`, площади помещений, привязку к УК.
- **УК-админ.** Создаю иерархию: ЖК → корпус → подъезд → этаж → квартира; для каждого подъезда определяю точку доступа (домофон, шлагбаум).
- **Диспетчер.** Просматриваю карту объектов своей УК на Yandex.Maps / 2GIS с маркером статуса (активный / на ОСС / приостановлен).
- **Сотрудник ОС.** При онбординге УК запускаю batch-импорт всех её МКД из ГИС ЖКХ.

### API (краткий перечень)

- `GET /objects` — список с фильтрами по региону, ОКТМО, УК, статусу.
- `POST /objects` — создание объекта.
- `POST /objects/import-from-gis-gkh` — batch-импорт МКД по списку ИНН УК.
- `POST /objects/{id}/sync-from-gis-gkh` — обновление одного объекта.
- `GET /objects/{id}/access-points` — список точек доступа.
- `POST /objects/{id}/access-points` — добавление точки.
- `PATCH /access-points/{id}` — изменение конфигурации.
- `GET /objects/{id}/premises` — список помещений.

### Ключевые сущности

`Object` (sharded by `id`), `Premise`, `AccessPoint`, `ObjectStatusHistory`. Связь с `Device` (модуль `device-svc`) через `AccessPoint.device_id`.

### REUSE / UPGRADE / NEW

- **REUSE.** Структура `backend/projects/` из `pass24-servicedesk` для CRUD-паттернов и FSM-статусов.
- **UPGRADE.** Citus-шардинг по `object_id` (горячая таблица).
- **NEW.** Импорт из ГИС ЖКХ через SOAP-обёртку (отдельный воркер с asyncio + Kafka topic `object.sync.requested`). Геопривязка к ФИАС GUID. Интеграция с PostGIS для поиска по координатам (партнёрское API B2B вызывает `/objects/lookup` по гео-координатам курьера).

---

## Модуль residents-svc — Жители и помещения

### Назначение

Реестр жителей объекта с привязкой к помещениям и долям владения (для расчёта кворума ОСС). Импорт из закрытой части ГИС ЖКХ возможен только через УК с её аккредитацией; согласие на ОПД — через договор управления (152-ФЗ ст. 6 п. 5 — обработка по договору без отдельного согласия).

### User stories

- **УК-админ.** Загружаю реестр жителей CSV-файлом (стандартный формат ГИС ЖКХ) или через API-pull, подтверждаю DPA-чекбоксом.
- **Житель.** При первом входе через ЕСИА система автоматически связывает мой ЕСИА-аккаунт с записью в реестре по совпадению ФИО + СНИЛС-хэш (без хранения СНИЛС в открытом виде).
- **УК-админ.** Меняю роль жителя «арендатор → собственник» при переоформлении квартиры.
- **Регуляторный комплаенс.** Запрашиваю отчёт по объёмам ПДн, согласиям и срокам хранения для аудита РКН.

### API (краткий перечень)

- `GET /residents` — список с фильтрами (объект, помещение, статус).
- `POST /residents/import` — батч-импорт CSV.
- `POST /residents` — добавление одного жителя.
- `PATCH /residents/{id}` — обновление.
- `DELETE /residents/{id}` — soft-delete (архив 7 лет по 152-ФЗ).
- `POST /residents/{id}/link-esia` — связь с ЕСИА-аккаунтом.
- `GET /residents/{id}/consents` — журнал согласий.

### Ключевые сущности

`Resident` (sharded by `object_id`), `ResidentConsent`, `ResidentRoleHistory`. Связь с `User` (identity-svc) и `Premise` (object-svc).

### REUSE / UPGRADE / NEW

- **REUSE.** Паттерны `backend/customers/` из `pass24-servicedesk` (CRUD по контактам).
- **UPGRADE.** Citus-шардинг по `object_id`. Шифрование ПДн at-rest на уровне колонок (pgcrypto).
- **NEW.** Связь с закрытой частью ГИС ЖКХ через УК-сессию (двусторонний SOAP с ЭЦП УК). Минимизация ПДн (не храним СНИЛС/паспорт — только хэш СНИЛС для матчинга и FIO). Журнал DPA-согласий с привязкой к версии договора управления.

---

## Модуль passes-svc — Электронный журнал пропусков (ядро системы)

### Назначение

Центральная подсистема: создание, выдача и отзыв пропусков 4 типов (разовый / временный / постоянный / курьерский «сквозной»). Все операции отражаются в `Pass` (горячая OLTP-таблица) и в `AccessEvent` ClickHouse (журнал проходов, append-only). Каждое событие подписывается XAdES-T (см. ADR-005).

### Типы пропусков

| Тип | Назначение | Срок | Кто выпускает | Каналы |
|---|---|---|---|---|
| **one_time** | Гость пришёл единократно | от 15 мин до 24 ч | житель | QR + BLE |
| **temporary** | Ремонт, няня, регулярный сервис | 1–30 дней | житель / УК | QR + BLE + NFC |
| **permanent** | Житель / член семьи / сотрудник | бессрочно (ежегодная проверка УК) | УК | BLE + NFC, мастер-ключ |
| **courier_through** | От B2B-партнёра (Yandex/OZON/WB) — «сквозной» ключ для нескольких точек подряд (шлагбаум + подъезд + лифт) | 30 мин — 2 ч | партнёр через API | QR + BLE |

### Статус-машина

```
draft → active → used (closed)
              → expired (TTL)
              → revoked (житель/УК/системой)
              → rejected (УК отклонила)
```

- Разовый пропуск автоматически переходит в `used` при первом успешном проходе.
- Временный — после `valid_until` → `expired`.
- Постоянный — пересмотр УК ежегодно; при удалении жителя из реестра → `revoked`.
- Курьерский «сквозной» — в одном `Pass` хранится массив `access_point_ids`, при первом успешном проходе по любой из точек статус = `active`, при истечении TTL без полного маршрута → `expired`.

### Бизнес-правила

- Лимиты по тарифу УК (4 уровня) и overage-ставки определены в NDA-материалах (`NDA/tariff-model.md`); технически они подгружаются в `billing-svc` из защищённого конфига и не хардкодятся в коде.
- При достижении 80 % любого жёсткого лимита — push УК-админу + e-mail + звонок CSM.
- При 100 % — soft-block новых пропусков на 24 ч + предложение апгрейда / overage.

### User stories

- **Житель.** В Госуслуги.Дом нажимаю «Заказать пропуск» → форма (имя гостя, телефон, время) → получаю QR + ссылку «отправить гостю в WhatsApp/Telegram».
- **Гость.** Получаю ссылку → открываю → вижу QR + инструкцию → подношу телефон к домофону → проход.
- **Курьер Yandex GO.** Заказ доставки требует доступа в подъезд → backend Yandex автоматически вызывает `POST /b2b/v1/keys/issue` → ESD создаёт `Pass` типа `courier_through` → ключ передан в приложение курьера.
- **УК-админ.** Просматриваю журнал пропусков по объекту, фильтрую по «отменённые», «использованные». Экспорт в XLSX.
- **Диспетчер.** Подтверждаю пропуски, требующие явного одобрения УК (например, для парковки).
- **Охранник.** В реальном времени вижу очередь подходящих гостей с фото из ЕСИА.

### API (краткий перечень)

- `GET /passes`, `POST /passes` — список и создание (роли: житель / УК / партнёр).
- `GET /passes/{id}`, `PATCH /passes/{id}` — детали и редактирование.
- `GET /passes/{id}/qr` — генерация QR-кода для отображения гостю.
- `POST /passes/{id}/share` — отправка ссылки гостю (deep link).
- `POST /passes/{id}/revoke` — отзыв.
- `POST /passes/{id}/approve` — одобрение УК.
- `POST /passes/{id}/reject` — отклонение УК.
- `GET /access-events` — журнал проходов (фильтр by `pass_id`, `object_id`, период).

### Ключевые сущности

`Pass` (sharded by `object_id`), `AccessEvent` (ClickHouse `access_events`, partitioned by month). Связь: `Pass.partner_id` (если выдан B2B), `Pass.access_point_ids[]`, `Pass.qr_token`, `Pass.ble_token_hash`.

### REUSE / UPGRADE / NEW

- **REUSE.** Паттерн `backend/tickets/` из `pass24-servicedesk` — статус-машина, comments, history, SLA-watcher (применимо для approval flow). 238 тестов как стартовая база.
- **UPGRADE.** Status machine выносим в отдельный класс на `transitions` (testability). SLA-watcher — из `lifespan` в Kafka consumer.
- **NEW.** XAdES-T-подпись на каждое `AccessEvent` (audit-svc). BLE-токены и QR-токены с TTL. Журнал в ClickHouse (вместо PostgreSQL — для RPS). Шардинг по `object_id`.

---

## Модуль acl-svc — Active Access Control (горячий путь)

### Назначение

Real-time проверка прав на проход в момент handshake ПАК-устройства с пользователем. Узкое место: пиковая нагрузка ~1 400 RPS утром и ~3 000 RPS в часы доставок, p99 latency должна быть < 100 ms (пользователь стоит у двери). Реализуется на Go (не Python) для минимизации latency.

### Контракт

Вход: `device_serial`, `access_point_id`, `pass_token` (BLE-handshake) или `partner_one_time_code`. Выход: `granted` / `denied` + `deny_reason`. Все запросы кэшируются Redis (TTL 5 мин для активных пропусков, инвалидация по Pub/Sub при revoke).

### User stories

- Пользователь подносит телефон → BLE-handshake → ПАК отправляет запрос в `acl-svc` → проверка действительности `Pass` + `valid_from <= now <= valid_until` + `access_point_id in Pass.access_point_ids` + статус не `revoked` → ответ `granted` за < 100 ms.
- При offline-режиме ПАК (нет связи) → локальная валидация по подписанному кэшу Ed25519 в EEPROM устройства (до 10 000 ключей).

### API

- `POST /acl/check` — горячая проверка (gRPC + Protocol Buffers для минимизации overhead).
- `GET /acl/keys/active` — синхронизация активных ключей для оффлайн-кэша устройства.

### REUSE / UPGRADE / NEW

- **REUSE.** Нет (нового слоя в `pass24-servicedesk` не было).
- **NEW.** Сервис целиком на Go. Redis-кэш ACL. Ed25519-подпись активного списка ключей для оффлайн-валидации устройством.

---

## Модуль oss-svc — Конструктор ОСС (общих собраний собственников)

### Назначение

Цифровой конструктор и проводник общих собраний собственников помещений МКД. Покрывает 10+ типовых решений: «Подключение УК к платформе ЕСД», «Установка ПАК», «Изменение тарифа УК», «Голосование по подрядчику ремонта», «Утверждение Service Desk-провайдера». Голосование — через Госуслуги.Дом с ЕСИА-подписью; кворум считается по площадям помещений из ГИС ЖКХ; итоговый протокол — XAdES-T-подписанный PDF.

### Жизненный цикл

```
draft → published → voting (10 дней по умолчанию)
                 → completed (если кворум достигнут)
                 → failed (если не достигнут)
                 → cancelled (отзыв инициатора)
```

### User stories

- **УК-админ.** Создаю ОСС из шаблона «Подключение к ЕСД» → wizard заполняет повестку → публикую → автоматическая рассылка всем жителям (push + email + Telegram).
- **Собственник с ≥ 10% долей.** Инициирую ОСС самостоятельно (по ЖК РФ).
- **Житель-собственник.** В Госуслуги.Дом вижу баннер «Активное ОСС в вашем доме» → кликаю → читаю повестку → голосую (за/против/воздержусь) → ЕСИА-подпись PKCS#7 detached → запись в `OSSVote.esia_signature`.
- **УК-админ.** По окончании голосования получаю итоговый PDF-протокол с XAdES-T от КриптоПро TSA + список всех голосов с верифицированными подписями + результат по кворуму.
- **Регулятор.** Запрашиваю верификацию протокола — подаю PDF в любой XAdES-T-валидатор → получаю подтверждение целостности и времени.

### API (краткий перечень)

- `GET /oss` — список ОСС объекта.
- `POST /oss` — создание из шаблона.
- `GET /oss/{id}` — детали.
- `POST /oss/{id}/publish` — публикация (старт голосования).
- `POST /oss/{id}/vote` — голос (требует ЕСИА-подпись).
- `GET /oss/{id}/results` — текущие результаты (после окончания).
- `GET /oss/{id}/protocol.pdf` — итоговый PDF с XAdES-T.
- `GET /oss/templates` — список шаблонов.

### Ключевые сущности

`OSSTemplate`, `OSS` (sharded by `object_id`), `OSSQuestion`, `OSSVote` (с ЕСИА-подписью), `OSSResult`, `OSSProtocol`. Long-running flow реализуется на Temporal.io (10-дневный workflow с автоматическим закрытием).

### REUSE / UPGRADE / NEW

- **REUSE.** Шаблоны и юр. содержание берётся из `knowledge-svc` (паттерн pass24-knowledge).
- **NEW целиком.** ЕСИА-голоса с PKCS#7 detached. Кворум по площадям (источник — ГИС ЖКХ через `object-svc`). XAdES-T от КриптоПро TSA. Temporal.io workflow (10-дневный жизненный цикл). PDF-генерация через WeasyPrint + подпись через КриптоПро HSM.

---

## Модуль uk-svc — Управляющие компании

### Назначение

Реестр УК-партнёров платформы. Хранит юридические реквизиты, лицензию по 191-ФЗ, договор с Оператором ЕСД, привязку объектов, billing-аккаунт, статус CSM. Источник реквизитов — DaData (поиск по ИНН) + проверка по реестру лицензий ГИС ЖКХ.

### User stories

- **Менеджер ОП.** Завожу нового лида (УК) → ввожу ИНН → DaData подтягивает реквизиты + проверка лицензии 191-ФЗ → создаётся `UK` со статусом `prospect`.
- **Юрист.** Согласую договор → ставлю флаг `contract_signed_at` → переход в `active`.
- **CSM-менеджер.** Просматриваю health-score УК (количество активных объектов, выручка, churn-сигналы), запускаю onboarding-чеклист.
- **Финбиллинг.** Меняю тариф УК с Базового на Стандартный после roll-out видеодомофонов.

### API (краткий перечень)

- `GET /uks` — список с фильтрами.
- `POST /uks` — добавление.
- `GET /uks/{id}` — детали.
- `PATCH /uks/{id}` — обновление.
- `POST /uks/lookup-by-inn` — поиск через DaData + ГИС ЖКХ.
- `GET /uks/{id}/health` — health-score CSM.
- `GET /uks/{id}/objects` — связанные объекты.

### Ключевые сущности

`UK`, `UKContract`, `UKLicense` (с TTL по `license_valid_until`). Связь с `Object`, `Subscription`.

### REUSE / UPGRADE / NEW

- **REUSE.** Паттерны `backend/customers/` из `pass24-servicedesk`, DaData-интеграция целиком (`backend/customers/dadata.py`).
- **UPGRADE.** Citus-шардинг по `id`.
- **NEW.** Проверка лицензии 191-ФЗ через ГИС ЖКХ. Health-score СSM на базе ClickHouse-агрегатов.

---

## Модуль partners-svc — Партнёрский шлюз B2B (Yandex / OZON / WB / СДЭК)

### Назначение

Публичный B2B API для крупных поставщиков заявок (курьерские службы, маркетплейсы). Партнёр получает «сквозной» одноразовый ключ доступа для своего курьера на конкретный объект и время доставки. Биллинг — по факту успешного открытия точки доступа (модель и ставки определены в `NDA/tariff-model.md`).

### Архитектура

```
[Yandex GO / OZON / WB]
        | (mTLS + JWT M2M Client Credentials)
        v
[Public B2B Gateway (Kong)]
        |
        v
[partners-svc]
        |
        +-> outbox: pass.created → Kafka topic 'partner.events'
        |
        +-> sync return ключа в ответе HTTP 200
```

### Сценарий выдачи ключа

1. Курьер подъезжает к МКД → приложение Yandex GO вызывает `POST /b2b/v1/keys/issue` с адресом и `partner_order_id`.
2. ESD `partners-svc`:
   - Проверяет квоту партнёра (`Partner.monthly_quota_keys`, `used_keys_current_month`).
   - Ищет `Object` по адресу/координатам через `object-svc`.
   - Применяет policy УК (временное окно, макс. курьеров/день, разрешённые подъезды).
   - Если требуется явное согласие жителя — push в моб.-кабинет, ожидание 60 сек, fallback `deny by default`.
   - Создаёт `Pass` типа `courier_through` через `passes-svc` с TTL 30 мин.
   - Возвращает `qr_token` + `ble_token` + deep-link.
3. Курьер открывает дверь → ESD получает `access_event` через `device-gateway` → отправляет webhook `pass.used` партнёру.

### Тарифные уровни доступа (структурно)

> Конкретные цены и пороги — в NDA-материалах (`NDA/tariff-model.md`), подтягиваются в `billing-svc` из конфига.

| Уровень | Доступ |
|---|---|
| **Read** | GET (status, lookup) |
| **RW + Webhooks** | issue/revoke + webhooks (биллинг по факту успешного открытия) |
| **Full Enterprise** | + custom rate-limit, dedicated SRE, custom DPA |

### User stories

- **Backend Yandex GO.** Получает M2M-токен, далее весь день вызывает `/keys/issue` для каждой доставки.
- **Партнёр (DevRel).** Подключается через sandbox, проходит UAT на 1 пилотном объекте, потом ramp-up rate-limit (10 → 100 → 1000 req/min за 14 дней).
- **CSM Оператора ЕСД.** Мониторит метрики партнёра (success rate, средняя latency, количество отказов из-за policy УК).

### API (краткий перечень)

- `POST /b2b/v1/keys/issue` — выдать разовый ключ.
- `GET /b2b/v1/keys/{key_id}` — статус.
- `POST /b2b/v1/keys/{key_id}/revoke` — отозвать.
- `POST /b2b/v1/objects/lookup` — поиск объекта по адресу/координатам.
- `POST /b2b/v1/access-events/subscribe` — подписаться на webhooks.
- `GET /b2b/v1/quota` — остаток квоты.

### Webhooks (callback на партнёра)

- `pass.issued` — ключ выпущен.
- `pass.used` — курьер прошёл.
- `pass.expired` — ключ истёк без использования.
- `pass.revoked` — ключ отозван (УК или системой).

### Ключевые сущности

`Partner`, `PartnerKey` (журнал выданных партнёрских ключей), `PartnerWebhookSubscription`, `PartnerUsageMetric`. Связь с `Pass` (`Pass.partner_id`).

### REUSE / UPGRADE / NEW

- **NEW целиком.** В `pass24-servicedesk` нет паттерна B2B-шлюза.
- mTLS-конфигурация через cert-manager в K8s. Rate-limit через Kong + Redis. ABAC-policies на уровне `partners-svc`.

---

## Модуль device-svc + device-gateway — ПАК (программно-аппаратный комплекс)

### Назначение

`device-svc` (Python/FastAPI) — реестр устройств, прошивок, конфигураций, мониторинг здоровья. `device-gateway` (Go) — горячий шлюз приёма heartbeat, access-attempts, телеметрии от ПАК через MQTT/WebSocket с mTLS. Вместе они обеспечивают onboarding производителей домофонов («Госуслуги.ДОМ Ready»), OTA-обновления прошивок, синхронизацию активных ключей для оффлайн-валидации.

### Аппаратная часть (ПАК-устройство)

- Контроллер на ESP32-S3 / Cortex-M4 + BLE 5.2 + Wi-Fi.
- Защищённый chip с ECC P-256 / GOST-ключевой парой.
- EEPROM 1 МБ — хранение 1000+ оффлайн-ключей.

### Программная часть (firmware)

- BLE-протокол: challenge-response с подписью ECDSA.
- Очередь оффлайн-проходов: до 10 000 событий локально, синхронизация при появлении сети.
- OTA: скачивание подписанного образа из `device-svc`, проверка signature перед flashing.
- Конфигурация: периодический pull `/devices/{serial}/config` (1×/час).
- mTLS-туннель к `device-gateway`, ключи rotated раз в 30 дней.

### User stories

- **OEM-производитель домофонов** (BAS-IP, SLINEX, Beward, Tantos). Регистрирует партию устройств: `POST /oem/v1/devices/register` с серийниками. Прошивает устройства предзагруженным cert-bundle.
- **Монтажник.** На объекте включает ПАК → устройство выполняет provisioning (`POST /oem/v1/devices/{serial}/provision`) → получает initial config + сертификат для mTLS.
- **Устройство.** Раз в 30 сек шлёт heartbeat (`POST /oem/v1/devices/{serial}/heartbeat`) с метриками (uptime, temperature, signal_strength, last_access_event_id).
- **УК-админ.** Запускает обновление прошивки на парк устройств → `device-svc` ставит задание в Kafka `device.firmware.update.requested` → устройства pull'ят при следующем heartbeat → подтверждают успех.

### API (краткий перечень) — OEM Gateway

- `POST /oem/v1/devices/register` — регистрация партии.
- `POST /oem/v1/devices/{serial}/provision` — initial provisioning.
- `POST /oem/v1/devices/{serial}/heartbeat` — heartbeat.
- `POST /oem/v1/devices/{serial}/access-attempt` — попытка прохода (получит ACL-решение).
- `POST /oem/v1/devices/{serial}/access-event` — фиксация прохода.
- `GET /oem/v1/devices/{serial}/firmware/check` — проверка наличия новой прошивки.
- `GET /oem/v1/devices/{serial}/firmware/{ver}` — скачивание (с проверкой подписи).
- `GET /oem/v1/devices/{serial}/config` — текущая конфигурация.
- `POST /oem/v1/devices/{serial}/logs/upload` — загрузка оффлайн-логов.
- `GET /oem/v1/devices/{serial}/keys/sync` — актуальный набор активных ключей (Ed25519-подписан).

### Ключевые сущности

`Device`, `DeviceFirmware`, `DeviceConfig`, `DeviceMetric` (TimescaleDB hypertable). Связь с `AccessPoint`, `Object`, `InstallerJob`.

### REUSE / UPGRADE / NEW

- **NEW целиком.** Нет аналога в `pass24-servicedesk`.
- `device-gateway` — отдельный сервис на Go (для mTLS + 5000 устройств × 1/30s heartbeat = ~170 RPS постоянной нагрузки).
- TimescaleDB hypertables для метрик с TTL 6 мес.
- OTA — паттерн image-based deploy из GitHub Actions `pass24-servicedesk` (REUSE для backend, NEW для firmware).

---

## Модуль installer-svc — Биржа монтажников и LMS

### Назначение

Маркетплейс сертифицированных монтажников ПАК с LMS (системой обучения), приёмочными тестами, рейтингами и комиссионной моделью. Заказчики — УК. Исполнители — независимые монтажники / монтажные компании. Структурно: стандартная комиссия Оператора ЕСД и повышенная за instant payout. Конкретные ставки — в `NDA/tariff-model.md`.

### User stories

- **Монтажник-новичок.** Регистрируюсь → прохожу LMS-курс «Монтаж ПАК Standard» (бесплатный, 6 модулей по 30 мин) → сдаю экзамен (порог 80%) → получаю сертификат с XAdES-T → попадаю в каталог биржи.
- **Монтажник-Pro.** Сдаю платный экзамен «Монтаж ПАК Pro» (стоимость сертификации — в `NDA/tariff-model.md`) → получаю расширенный допуск.
- **УК-админ.** Создаю заявку на монтаж → биржа автоматически назначает монтажника по гео + рейтингу или открывает аукцион.
- **Монтажник.** В мобильном приложении вижу новое задание → принимаю → в назначенный день делаю фото-обследование → монтаж → запускаю приёмочный тест (10 точек: ЕСИА → BLE → открытие → лог) → закрываю.
- **L2-инженер ESD.** Подтверждаю приёмочный тест по фото и логам устройства.

### API (краткий перечень) — Installer BFF

- `GET /installer/jobs` — мои задания.
- `POST /installer/jobs/{id}/start` — начать работу.
- `POST /installer/jobs/{id}/test` — прогнать приёмочный тест.
- `POST /installer/jobs/{id}/finish` — закрыть задание (с фото/видео-актом).
- `GET /installer/lms/courses` — доступные курсы.
- `POST /installer/lms/courses/{id}/exam` — сдать экзамен.
- `GET /installer/marketplace/jobs` — открытые задания биржи.
- `POST /installer/marketplace/jobs/{id}/bid` — подать заявку.

### Ключевые сущности

`Installer`, `InstallerCertification`, `InstallerJob`, `Course`, `Lesson`, `ExamAttempt`, `Bid`, `MarketplaceJob`.

### REUSE / UPGRADE / NEW

- **REUSE.** Knowledge-модуль `pass24-servicedesk` для контента курсов LMS.
- **UPGRADE.** Geo-маршрутизация заданий (PostGIS).
- **NEW.** Биржа целиком, рейтинговая система, комиссии, выплаты (интеграция с Тинькофф Бизнес / эквайринг).

---

## Модуль billing-svc — Биллинг, тарифы, лимиты УК

### Назначение

Реализует тарифную модель ESD (см. ADR-003). Наследник модели PASS24 ADR-001 «жёсткие лимиты с апгрейдом». Покрывает: SaaS-подписки УК (4 уровня), разовую продажу ПАК (CAPEX), B2B-подписки партнёров (3 уровня), комиссии биржи монтажа, доплаты (SMS, голосовой ассистент, white-label, доп. точки). **Конкретные тарифы, лимиты, ставки комиссий и overage — в `NDA/tariff-model.md`** и подгружаются в `billing-svc` из защищённого конфига; в коде хардкод не допускается.

### User stories

- **Финбиллинг.** Закрываю расчётный период → автоматически выставляются счёта всем УК (СБИС / Контур.Диадок).
- **УК-админ.** Вижу в кабинете «Использовано N / лимит пропусков» по своему тарифу → push-уведомление при достижении 80 % → опция «апгрейд за 5 минут».
- **УК-админ.** Получаю автоматическое уведомление об истечении лицензии 191-ФЗ за 30 дней — с предложением продлить через интегрированный сервис.
- **Биржа монтажа.** При завершении задания комиссия (ставка из конфига) автоматически удерживается с выплаты монтажнику.

### API (краткий перечень)

- `GET /billing/subscription` — текущий тариф/лимиты.
- `POST /billing/upgrade` — запрос апгрейда.
- `GET /billing/usage` — использование (real-time).
- `GET /billing/invoices` — счета.
- `POST /billing/invoices/{id}/pay` — оплата (Yandex Касса / Тинькофф Касса).
- `GET /billing/overage` — overage-счётчики.

### Ключевые сущности

`BillingAccount`, `Subscription`, `UsageMetric`, `Overage`, `Invoice`, `Payment`. Связь с `UK`, `Object`, `Partner`.

### REUSE / UPGRADE / NEW

- **REUSE.** Тарифная модель из `pass24-knowlege-base` (Тарифы PASS24 v1.1) — структура, лимиты, апгрейды.
- **NEW целиком.** Нет в `pass24-servicedesk`. Эквайринг (Yandex Касса / Тинькофф), генерация счетов через Контур.Диадок.

---

## Модуль assistant-svc — AI-ассистент жителя и УК (RAG)

### Назначение

Интеллектуальный чат-ассистент с RAG (Retrieval-Augmented Generation) поверх базы знаний. LLM — Claude Sonnet 4 (Anthropic) через прокси с RU-фильтрацией. Векторная база — Qdrant Cluster. Локальный fallback на YandexGPT / GigaChat для регуляторных сценариев на случай блокировок Claude.

### User stories

- **Житель.** Спрашиваю «Как заказать пропуск курьеру?» → ассистент отвечает с пошаговой инструкцией + ссылками на статьи KB.
- **Житель.** «Что такое ОСС?» → ассистент объясняет в режиме «как соседу» + предлагает проголосовать в активном ОСС.
- **УК-админ.** «Как создать ОСС по подключению ПАК?» → ассистент даёт чек-лист + ссылки на шаблоны решений.
- **Диспетчер.** «Как обработать массовый поток пропусков в новогодние праздники?» → ассистент даёт инструкцию + ссылку на playbook.
- **Монтажник.** «Какой код ошибки `E_BLE_AUTH_FAILED`?» → ассистент объясняет + ссылается на инструкцию по перепрошивке.

### API

- `POST /assistant/chat` — запрос к ассистенту (с историей).
- `POST /assistant/feedback` — оценка ответа (для улучшения).

### REUSE / UPGRADE / NEW

- **REUSE 1-в-1.** Модуль `backend/assistant/` из `pass24-servicedesk` (Claude Sonnet 4 + Qdrant + RAG-pipeline).
- **UPGRADE.** Кластеризация Qdrant (3 ноды). Локальный fallback на YandexGPT / GigaChat.
- **NEW.** Регуляторные сценарии (ответы про 152-ФЗ, 191-ФЗ, ЖК РФ). ОСС-help. Ассистент монтажника по кодам ошибок ПАК.

---

## Модуль stats-svc — Аналитика и отчёты

### Назначение

Витрина OLAP-аналитики и отчётности для УК, Минцифры, Минстроя, инвесторов. Источник — ClickHouse (журнал `access_events`, агрегаты по биллингу) + PostgreSQL (актуальное состояние). Визуализация — Metabase / Apache Superset для self-service. PDF/XLSX-отчёты по cron — через `notifications-svc`.

### User stories

- **УК-админ.** Дашборд «Проходы по объектам за неделю»: ECharts-график, разбивка курьеры/гости/жители, нагрузка на диспетчера.
- **Минцифры (комплаенс-аудитор).** Запрос «Количество подключённых УК и объектов по регионам» → отчёт PDF с XAdES-T.
- **Инвестор.** Self-service дашборд в Superset: MRR, ARR, churn УК, LTV объекта, unit-economics.
- **CFO.** Cohort-анализ УК по тарифам, прогноз выручки на 6 мес.

### API (краткий перечень)

- `GET /stats/overview` — summary для роли.
- `POST /stats/reports/run` — генерация отчёта.
- `GET /stats/reports/{id}/download` — скачивание.
- `GET /stats/dashboards` — список доступных дашбордов.

### REUSE / UPGRADE / NEW

- **REUSE.** Модуль `backend/stats/` из `pass24-servicedesk` (REST API для дашбордов) + ECharts во фронтенде.
- **NEW.** ClickHouse OLAP-движок. Metabase / Apache Superset для self-service. ETL через Apache Airflow (PostgreSQL → ClickHouse).

---

## Модуль notifications-svc — Уведомления (multi-channel)

### Назначение

Мультиканальная доставка уведомлений: Push в Госуслуги.Дом (главный канал по ADR-004), SMTP, SMS, Telegram, WhatsApp Business API. Шаблонизатор — Jinja2 (как в pass24) с версионированием и A/B-тестами. Воркер уведомлений — отдельный consumer Kafka topic `notifications.send`.

### User stories

- **Житель.** Получаю push в Госуслуги.Дом «Активное ОСС в вашем доме, проголосуйте до 5 мая».
- **УК-админ.** Получаю email-сводку «Превышен лимит пропусков на объекте ЖК Заречье на 80%».
- **Курьер Yandex GO.** Получаю SMS с deep-link на ключ доступа (резерв на случай отказа push).
- **Сотрудник ОС.** Получаю Telegram-уведомление о новом критическом тикете.

### Каналы

- **Push в Госуслуги.Дом** — закрытый канал, требует партнёрского соглашения с ПАО МРТС (см. `regulatory-roadmap.md`, ADR-004).
- **SMTP** — Yandex 360 / VK WorkMail (РФ-локализация).
- **SMS** — СМС-Центр / SMS-Aero / SMSRU. Тарифицируется отдельно как доплата ко всем тарифам (ставка — в `NDA/tariff-model.md`).
- **Telegram** — собственный бот @ESDOperatorBot (см. модуль `telegram-svc`).
- **WhatsApp Business** — через сертифицированного провайдера (Wazzup24 / 1Sender).

### API (краткий перечень)

- `POST /notifications/send` — отправка (внутренний API, вызывается другими сервисами через Kafka).
- `GET /notifications/preferences` — настройки пользователя.
- `PUT /notifications/preferences` — изменение.
- `GET /notifications/history` — история.

### REUSE / UPGRADE / NEW

- **REUSE.** Структура `backend/notifications/` из `pass24-servicedesk`, Jinja2-шаблоны.
- **UPGRADE.** Kafka consumer `notifications.send` вместо `lifespan`-таска. DLQ + ретраи (max 5 попыток с exp backoff).
- **NEW.** Push в Госуслуги.Дом (через Partner API ПАО МРТС). WhatsApp Business канал.

---

## Модуль telegram-svc — Telegram-бот УК и диспетчера

### Назначение

Корпоративный Telegram-бот для сотрудников УК / диспетчеров / монтажников: получение уведомлений, ответы на тикеты, вызов AI-ассистента. Реализуется на aiogram 3 + self-hosted Telegram Bot API (для файлов > 20 МБ).

### User stories

- **УК-админ.** Привязываю бота через QR-код в кабинете → получаю уведомления о критических событиях.
- **Диспетчер.** Из бота отвечаю на запрос жителя через wizard.
- **Монтажник.** Через бота вижу новые задания биржи + могу принять на ходу.

### REUSE / UPGRADE / NEW

- **REUSE 1-в-1.** Модуль `backend/telegram/` из `pass24-servicedesk` (aiogram 3 + handlers + middlewares + services). Гайд `telegram-bot-api-self-hosted.md` — переиспользуем.
- **UPGRADE.** Вынос `lifespan dispatcher` в отдельный микросервис с Kafka consumer.

---

## Модуль tickets-svc — Service Desk монтажа и саппорта УК

### Назначение

Внутренний Service Desk для команды Оператора ЕСД и монтажных бригад. Покрывает инциденты установки ПАК, обращения УК по техническим вопросам, эскалации L1 → L2 → инженер. SLA по бизнес-часам (см. `pass24-servicedesk/agent_docs/guides/sla.md`).

### User stories

- **L1-оператор.** Принимаю звонок УК → создаю тикет → классификация по ключевым словам → попытка self-heal через AI-ассистента → если не закрыто за 5 мин — эскалация L2.
- **L2-инженер.** Удалённое подключение к ПАК (если поддерживается), анализ BLE-логов, переконфигурация.
- **Полевой инженер.** Получаю задание в мобильное приложение → выезжаю → закрываю с фото-актом.
- **Менеджер ОС.** Просматриваю CSAT-метрики команды.

### REUSE / UPGRADE / NEW

- **REUSE 1-в-1.** Модуль `backend/tickets/` из `pass24-servicedesk` целиком — модели `Ticket`, `Comment`, `Attachment`, статус-машина, SLA-watcher, CSAT scheduler. 238 тестов.
- **UPGRADE.** Привязка тикета к `Object` + `Device` + `Installer` (вместо только `Customer`). SLA-watcher из `lifespan` в Kafka consumer + Redis Streams.
- **NEW.** Эскалация в биржу монтажа для полевого выезда.

---

## Модуль knowledge-svc — База знаний и LMS

### Назначение

Библиотека статей FAQ, инструкций, методичек ОСС, юридических справочников + LMS для монтажников и УК-сотрудников. Источник материалов для AI-ассистента (RAG).

### Категории под госконтур

- ОСС: шаблоны решений, методички, FAQ.
- Подключение УК: чек-листы, регламенты, видео-инструкции.
- Подача в Минцифры / ГИС ЖКХ: как стать «Commercial IS».
- Юридические FAQ: 152-ФЗ, 191-ФЗ, ЖК РФ.
- Антитеррор: законодательство и регламенты.
- Курсы LMS: 6+ курсов для монтажников ПАК (Lite / Standard / Pro / Enterprise).

### API

- `GET /knowledge/articles` — список с фильтрами.
- `GET /knowledge/articles/{slug}` — статья.
- `POST /knowledge/search` — full-text + векторный поиск.
- `POST /knowledge/articles/{id}/feedback` — лайк/дизлайк.
- `GET /knowledge/lms/courses` — курсы.

### REUSE / UPGRADE / NEW

- **REUSE 1-в-1.** Модуль `backend/knowledge/` из `pass24-servicedesk`.
- **NEW.** LMS-функционал, сертификация с XAdES-T, категории под госконтур.

---

## Модуль audit-svc — Immutable audit log

### Назначение

Подписанный append-only журнал юридически значимых событий: каждое `AccessEvent`, каждый голос ОСС, каждое изменение тарифа УК, каждый отзыв пропуска. Подпись — XAdES-T от КриптоПро HSM 2.0 + timestamp от КриптоПро TSA. Используется для соответствия ФСТЭК-17 (контроль целостности — ОЦЛ) и для юридической силы протоколов ОСС.

### User stories

- **Регулятор / комплаенс-аудитор.** Получаю CSV/XML-выгрузку событий за период с XAdES-T-подписью оператора → могу верифицировать целостность независимо.
- **Юрист УК.** Запрашиваю audit-trail по конкретному ОСС → получаю подписанный PDF с историей всех голосов и подписей.
- **CISO.** Расследую инцидент → запрашиваю все действия пользователя за период.

### API

- `POST /audit/events` — запись (внутренний API, вызывается всеми сервисами через Kafka).
- `GET /audit/events` — поиск с фильтрами.
- `GET /audit/events/export` — экспорт (CSV/XML/PDF) с XAdES-T-подписью.
- `POST /audit/verify` — верификация подписи внешнего файла.

### REUSE / UPGRADE / NEW

- **NEW целиком.** Нет аналога в `pass24-servicedesk`.
- ClickHouse `audit_log` (append-only).
- HSM-пул КриптоПро HSM 2.0 + очередь Kafka topic `audit.sign` (CPU-bound операция, не блокируем основной поток).

---

## Связанные документы

- `architecture.md` — архитектура системы и инфраструктуры.
- `adr.md` — архитектурные решения (ADR-002 стек, ADR-003 тарифы, ADR-004 Госуслуги.Дом-only, ADR-005 XAdES-T audit, ADR-006 RBAC).
- `api-map.md` — полная карта API по 5 каналам.
- `data-model.md` — модель данных (PostgreSQL + ClickHouse).
- `functional-matrix.md` — матрицы Роли × Модули и Модули × Этапы.
- `load-profile.md` — нефункциональные требования (RPS, кэш, шардинг, observability).
- `NDA/tariff-model.md` (вне репозитория) — детальная тарифная модель.
- `operations-playbooks.md` — операционные плейбуки (онбординг УК, монтаж ПАК, инциденты, B2B).
- `regulatory-roadmap.md` — регуляторный трек (ЕСИА, ФСТЭК, ИС ГИС ЖКХ, СМЭВ-4).
- `risks-register.md` — реестр рисков (без финансовых деталей).
- `NDA/team-plan.md` (вне репозитория) — план команды и RACI.
