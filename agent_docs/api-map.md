# Карта API ESD ServiceDesk

> Полная карта внешних и внутренних API платформы Оператора ЕСД, разнесённая по 5 каналам потребления + интеграции с ГИС ЖКХ. Аутентификация: ЕСИА OAuth 2.0 / OIDC для конечных пользователей; JWT RS256 + Vault rotation внутри кластера; M2M Client Credentials + mTLS для B2B-партнёров и OEM-производителей; mTLS-сертификаты с криптопроцессоров — для ПАК-устройств. Подробное ТЗ модулей — в `functional-spec.md`. Регуляторный контекст — в `regulatory-roadmap.md` и ADR-004.

---

## 1. BFF Mobile — встраивание в Госуслуги.Дом

> Endpoint: `https://api.esd.gov.ru/mobile/v1/`. Доступ через партнёрское соглашение с Оператором приложения (ПАО МРТС) и СМЭВ-4. Auth: ЕСИА-токен жителя (с claims `oid`, `snils_hash`).

| Endpoint | Method | Описание | Роль |
|---|---|---|---|
| `/passes` | GET | Список моих пропусков | RESIDENT |
| `/passes` | POST | Создать пропуск (one_time / temporary / permanent) | RESIDENT |
| `/passes/{id}` | GET | Детали пропуска | RESIDENT |
| `/passes/{id}` | PATCH | Изменить временное окно / список точек | RESIDENT |
| `/passes/{id}/qr` | GET | QR-код для отображения гостю | RESIDENT |
| `/passes/{id}/share` | POST | Отправить ссылку гостю (deep-link с временным токеном) | RESIDENT |
| `/passes/{id}/cancel` | POST | Отменить пропуск | RESIDENT |
| `/access-events` | GET | Журнал моих проходов (свои + гостей) | RESIDENT |
| `/access-points` | GET | Доступные точки доступа в моём МКД | RESIDENT |
| `/oss/active` | GET | Активные ОСС в моём МКД | RESIDENT |
| `/oss/{id}/vote` | POST | Голосовать (с ЕСИА-подписью PKCS#7 detached) | RESIDENT |
| `/assistant/ask` | POST | Запрос AI-ассистенту (RAG поверх KB) | RESIDENT |
| `/notifications/preferences` | GET / PUT | Настройки уведомлений (push / SMS / email) | RESIDENT |
| `/profile/uk` | GET | Информация о моей УК | RESIDENT |
| `/profile/object` | GET | Информация о моём объекте (адрес, точки доступа) | RESIDENT |

---

## 2. BFF Web Operator — кабинеты УК / Диспетчер / Охрана

> Endpoint: `https://app.esd.gov.ru/operator/api/`. Auth: ЕСИА-токен сотрудника + JWT с claims (`role`, `uk_id`, `object_ids`).

| Endpoint | Method | Описание | Роль(и) |
|---|---|---|---|
| `/operator/objects` | GET | Список объектов УК | PROPERTY_MANAGER |
| `/operator/objects/{id}` | GET / PATCH | Детали и редактирование | PROPERTY_MANAGER |
| `/operator/objects/{id}/passes` | GET | Журнал пропусков по объекту | PROPERTY_MANAGER, DISPATCHER |
| `/operator/passes/{id}/approve` | POST | Подтвердить пропуск | DISPATCHER |
| `/operator/passes/{id}/reject` | POST | Отклонить пропуск | DISPATCHER |
| `/operator/access-events` | GET | Real-time журнал проходов с фильтром | PROPERTY_MANAGER, SECURITY_GUARD |
| `/operator/residents` | GET | Жители объекта | PROPERTY_MANAGER |
| `/operator/residents/import` | POST | Batch-импорт CSV из ГИС ЖКХ | PROPERTY_MANAGER |
| `/operator/residents/{id}/grant-permanent` | POST | Выдать постоянный ключ жителю | PROPERTY_MANAGER |
| `/operator/devices` | GET | Список ПАК-устройств объекта | PROPERTY_MANAGER |
| `/operator/devices/{id}/firmware/update` | POST | Триггер OTA-обновления | PROPERTY_MANAGER, ADMIN |
| `/operator/oss` | POST | Создать ОСС из шаблона | PROPERTY_MANAGER |
| `/operator/oss/{id}/results` | GET | Результаты ОСС | PROPERTY_MANAGER |
| `/operator/oss/{id}/protocol.pdf` | GET | Итоговый PDF с XAdES-T | PROPERTY_MANAGER |
| `/operator/billing/subscription` | GET | Текущий тариф/лимиты | PROPERTY_MANAGER |
| `/operator/billing/upgrade` | POST | Запрос апгрейда тарифа | PROPERTY_MANAGER |
| `/operator/installers/marketplace` | GET | Каталог биржи монтажников | PROPERTY_MANAGER |
| `/operator/tickets` | GET / POST | Service Desk монтажа и поддержки | PROPERTY_MANAGER, SUPPORT_AGENT |
| `/operator/reports/run` | POST | Сгенерировать отчёт | PROPERTY_MANAGER |
| `/operator/reports/{id}/download` | GET | Скачать отчёт (PDF/XLSX) | PROPERTY_MANAGER |
| `/operator/audit-log` | GET | Аудит действий по объекту | PROPERTY_MANAGER, COMPLIANCE_AUDITOR |

---

## 3. Public B2B Gateway — Yandex GO / OZON / WB / СДЭК

> Endpoint: `https://api.esd.gov.ru/b2b/v1/`. Auth: M2M JWT (Client Credentials grant) + mTLS клиентским сертификатом партнёра. Rate-limit per `Partner.api_tier`.

| Endpoint | Method | Описание | Роль |
|---|---|---|---|
| `/keys/issue` | POST | Запросить разовый «сквозной» ключ для курьера (`pass_type=courier_through`, TTL 30 мин) | PARTNER_API (RW+, Full) |
| `/keys/{key_id}` | GET | Статус ключа | PARTNER_API (Read+) |
| `/keys/{key_id}/revoke` | POST | Отозвать ключ | PARTNER_API (RW+, Full) |
| `/objects/lookup` | POST | Поиск объекта по адресу/координатам (PostGIS) | PARTNER_API (Read+) |
| `/access-events/subscribe` | POST | Подписка на webhooks | PARTNER_API (RW+, Full) |
| `/access-events/webhook-test` | POST | Проверка доставки webhook | PARTNER_API (RW+, Full) |
| `/quota` | GET | Остаток квоты ключей в текущем месяце | PARTNER_API (все) |

### 3.1. Webhooks (callback на партнёра)

> Партнёр регистрирует `webhook_url` через `/access-events/subscribe`. ESD отправляет HTTPS POST с заголовком `X-ESD-Signature` (HMAC-SHA256). Ретраи: 5 попыток с exp backoff (5 с → 10 м), DLQ при окончательном отказе.

| Event | Когда отправляется | Payload (ключевые поля) |
|---|---|---|
| `pass.issued` | Ключ выпущен через `/keys/issue` | `pass_id`, `partner_order_id`, `qr_token`, `valid_until` |
| `pass.used` | Курьер прошёл (получено `access_event` от ПАК) | `pass_id`, `partner_order_id`, `access_point_id`, `timestamp` |
| `pass.expired` | TTL истёк без использования | `pass_id`, `partner_order_id`, `expired_at` |
| `pass.revoked` | УК или система отозвали ключ | `pass_id`, `partner_order_id`, `revoke_reason`, `revoked_at` |

---

## 4. OEM Gateway — «Госуслуги.ДОМ Ready» (производители домофонов)

> Endpoint: `https://api.esd.gov.ru/oem/v1/`. Auth: M2M JWT + mTLS клиентским сертификатом производителя. Подключение по программе сертификации (BAS-IP, SLINEX, Beward, Tantos, BOLID, Sigur и др.).

| Endpoint | Method | Описание | Роль |
|---|---|---|---|
| `/devices/register` | POST | Регистрация партии устройств (серийники, модель) | OEM_PARTNER |
| `/devices/{serial}/provision` | POST | Initial provisioning при первой загрузке | DEVICE |
| `/devices/{serial}/heartbeat` | POST | Heartbeat (uptime, signal, temperature) — каждые 30 с | DEVICE |
| `/devices/{serial}/access-attempt` | POST | Попытка прохода (получит ACL-решение `granted`/`denied`) | DEVICE |
| `/devices/{serial}/access-event` | POST | Зафиксированный проход (immutable, идёт в `audit-svc`) | DEVICE |
| `/devices/{serial}/firmware/check` | GET | Проверить наличие новой прошивки | DEVICE |
| `/devices/{serial}/firmware/{ver}` | GET | Скачать прошивку (с проверкой signature) | DEVICE |
| `/devices/{serial}/config` | GET | Текущая конфигурация устройства | DEVICE |
| `/devices/{serial}/logs/upload` | POST | Загрузить оффлайн-логи | DEVICE |
| `/devices/{serial}/keys/sync` | GET | Актуальный набор активных ключей (Ed25519-подписан) для оффлайн-валидации | DEVICE |

---

## 5. Installer BFF — мобильное приложение монтажника

> Endpoint: `https://app.esd.gov.ru/installer/api/`. Auth: ЕСИА-токен монтажника + JWT с claims (`installer_id`, `certifications`).

| Endpoint | Method | Описание | Роль |
|---|---|---|---|
| `/installer/jobs` | GET | Мои монтажные задания | INSTALLER |
| `/installer/jobs/{id}/start` | POST | Начать работу (фиксация старта) | INSTALLER |
| `/installer/jobs/{id}/test` | POST | Прогнать приёмочный тест (10 точек) | INSTALLER |
| `/installer/jobs/{id}/finish` | POST | Закрыть задание (фото/видео-акт) | INSTALLER |
| `/installer/lms/courses` | GET | Доступные мне курсы | INSTALLER |
| `/installer/lms/courses/{id}/exam` | POST | Сдать экзамен (порог 80%) | INSTALLER |
| `/installer/marketplace/jobs` | GET | Открытые задания биржи | INSTALLER |
| `/installer/marketplace/jobs/{id}/bid` | POST | Подать заявку на задание | INSTALLER |

---

## 6. Внутренние интеграции с ГИС ЖКХ (SOAP)

> Direction: ESD → ГИС ЖКХ. Аккредитация — «Commercial IS» (см. `regulatory-roadmap.md` шаги 5–8). Подпись запросов: PKCS#7 detached по GOST Р 34.10-2012, формат XAdES-T, базовый URL `https://api.dom.gosuslugi.ru/`.

### Используемые SOAP-эндпоинты ГИС ЖКХ

| Сервис | URL | Что используем | Частота |
|---|---|---|---|
| `/Nsi/` | http://open-gkh.ru/Nsi/ | Справочники: ОКТМО, регионы, типы помещений, виды работ ЖКУ | 1×/сутки cron |
| `/NsiBase/` | http://open-gkh.ru/NsiBase/ | Базовые НСИ (классификаторы услуг, типы данных) | 1×/неделю cron |
| `/PremisesBase/` | http://open-gkh.ru/PremisesBase/ | Реестр помещений (общая часть): площади, привязка к МКД | по запросу при онбординге УК |
| `/Inspection/` | http://open-gkh.ru/Inspection/ | Контроль и надзор (для отчётности Минцифры) | 1×/мес cron |
| `/NsiService/` | http://open-gkh.ru/NsiService/ | Асинхронный экспорт больших справочников | по требованию |

### Внутренние ESD endpoints для управления синхронизацией

| Endpoint | Method | Описание | Роль |
|---|---|---|---|
| `/admin/gis-gkh/sync/objects` | POST | Запустить синхронизацию МКД | ADMIN |
| `/admin/gis-gkh/sync/uks` | POST | Запустить синхронизацию реестра УК | ADMIN |
| `/admin/gis-gkh/sync/nsi` | POST | Обновить нормативно-справочную информацию | ADMIN |
| `/admin/gis-gkh/health` | GET | Статус интеграции (latency, error rate, last sync) | ADMIN, CISO |
| `/admin/gis-gkh/sync-jobs/{id}` | GET | Статус конкретной asynchronous-задачи | ADMIN |

### Cron-джобы (в `gis-gkh-sync` worker)

| Job | Расписание | Что делает |
|---|---|---|
| `sync-nsi-base` | Раз в неделю, понедельник 03:00 МСК | Полное обновление базовых справочников |
| `sync-nsi-incremental` | Каждые 6 часов | Инкрементальное обновление по `lastModifiedDate` |
| `sync-uk-registry` | Раз в сутки, 04:00 МСК | Реестр УК (через ОГВ-открытые данные) + проверка лицензий 191-ФЗ |
| `sync-mkd-registry` | По требованию (при онбординге УК) | Реестр МКД конкретной УК через `PremisesBase` |
| `inspection-report-export` | 1-го числа месяца, 06:00 МСК | Выгрузка отчётности для надзора |

---

## 7. Внутренние ESD интеграции с реестрами и сервисами

| Внешний сервис | Назначение | Канал | Частота |
|---|---|---|---|
| ФИАС / ГАР (`fias.nalog.ru`) | Адресная нормализация, FIAS GUID | Загрузка GAR-XML 1×/мес + REST API | по требованию + 1×/мес |
| DaData | Поиск по ИНН, обогащение реквизитов УК | REST API | по запросу |
| Bitrix24 / amoCRM | CRM-синхронизация лидов УК | Webhook + REST | real-time |
| КриптоПро TSA | Timestamp для XAdES-T | TSP-протокол (RFC 3161) | при каждой подписи audit-event |
| КриптоПро HSM 2.0 | Хранение корневых ключей оператора | PKCS#11 | при каждой подписи |
| Yandex.Maps / 2GIS | Геокодинг и карты | REST | по запросу |

---

## Связанные документы

- `architecture.md` — архитектурные слои, API Gateway (Kong), Service Mesh (Istio), Device Gateway.
- `functional-spec.md` — детальное ТЗ модулей за каждым каналом API.
- `data-model.md` — модели данных, на которых работают эндпоинты.
- `load-profile.md` — нагрузочные таргеты по каналам (5 000 RPS sustained, 15 000 burst).
- `regulatory-roadmap.md` — путь к аккредитации «Commercial IS» и партнёрскому API ЕПГУ через СМЭВ-4.
- `adr.md` — ADR-004 про жёсткую стратегию Госуслуги.Дом-only канала.
