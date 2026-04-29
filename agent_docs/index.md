# Навигация по документам

> Карта **технической** документации Оператора «Единой Среды Доступа» (ЕСД). Читать только релевантные файлы — каждый документ кратко описан ниже.
>
> **Финансовая часть** (тарифная модель, финансовая модель, инвестиционный план, план команды и ФОТ) находится **вне репозитория** в папке `NDA/` — она исключена через `.gitignore` и доступна только рабочей группе и C-level. В этом проекте мы работаем только с разработчиками, поэтому в технических документах конкретные суммы, тарифы и оценочные показатели не указываются.

## Стратегия и контекст

- `../AGENTS.md` — описание проекта, цели команды, контекст и ограничения. **Читать первым.**
- `architecture.md` — архитектура продукта (верхний уровень) + высоконагруженная архитектура серверной платформы ESD ServiceDesk (микросервисы, инфраструктура, REUSE/UPGRADE/NEW vs pass24-servicedesk).
- `adr.md` — архитектурные решения. Шесть записей: ADR-001 (запуск ЕСД отдельным юрлицом), ADR-002 (стек), ADR-003 (тарифная модель), ADR-004 (стратегия Госуслуги.Дом-only), ADR-005 (audit XAdES-T), ADR-006 (RBAC и роли).

## Функциональное ТЗ

- `functional-spec.md` — детальное ТЗ к 18 микросервисам платформы (назначение, user stories, API endpoints, сущности, REUSE/UPGRADE/NEW).
- `functional-matrix.md` — две матрицы: Роли × Модули (CRUDX) и Модули × Этапы roadmap.
- `api-map.md` — карта API по 5 каналам (BFF Mobile / BFF Web Operator / Public B2B Gateway / OEM Gateway / Installer BFF) + интеграция с ГИС ЖКХ SOAP + webhooks для партнёров.
- `data-model.md` — модель данных PostgreSQL (25+ сущностей) + ClickHouse (OLAP), стратегия шардинга и архивации.

## Нагрузка, наблюдаемость, верификация

- `load-profile.md` — нефункциональные требования под high-load (RPS/IOPS-расчёт на 24 мес, пики, кэш, шардинг, georesilience, observability stack, SLI/SLO, стратегия тестирования).

## Команда и операции

- `operations-playbooks.md` — 5 операционных плейбуков: подключение УК, установка ПАК, обработка инцидента «не открылся проход», подключение B2B-партнёра, поток заявки от партнёра до открытия точки доступа.

> **Вне репозитория (`NDA/`):** `team-plan.md` (план вывода команды, FTE, ФОТ, RACI), `tariff-model.md` (тарифная модель ESD: 4 уровня УК + 3 уровня B2B-API + ПАК CAPEX + биржа), `financial-model.md` (целевые KPI revenue/EBITDA/оценки, unit-экономика), `investment-plan.md` (бюджет, инвестиции, runway, bridge-round).

## Регуляторика и риски

- `regulatory-roadmap.md` — регуляторный трек (ЕСИА / ФСТЭК-17 / Commercial IS / СМЭВ-4 / 152-ФЗ / лицензирование ПАК) с шагами по месяцам.
- `risks-register.md` — сводный реестр рисков (технические + HR + коммерческие + регуляторные + репутационные + финансовые) с trip-wires и планом эскалации.

## Журнал работы

- `development-history.md` — журнал итераций проекта (последние 10 записей; архив — `development-history-archive.md`).

## Гайды и шаблоны (без изменений)

- `guides/dod.md` — критерии завершенности (DoD).
- `guides/environment-setup.md` — настройка окружения; применять при инициализации проекта.
- `guides/logging.md` — логирование скриптов/интеграций.
- `guides/archiving-and-temp.md` — архивация и временные файлы.
- `templates/architecture.md`, `templates/adr.md`, `templates/development-history.md` — шаблоны.

## Порядок чтения для разных ролей

| Роль | Что читать |
|---|---|
| Новый разработчик | `AGENTS.md` → `architecture.md` → `adr.md` → `functional-spec.md` → `data-model.md` → `api-map.md` |
| Архитектор / Tech Lead | `architecture.md` + `load-profile.md` + `adr.md` + `data-model.md` |
| Продакт / PM | `architecture.md` (раздел Roadmap) + `functional-matrix.md` + `operations-playbooks.md` (тарифная модель — в `NDA/tariff-model.md`) |
| Юрист / Compliance | `regulatory-roadmap.md` + `adr.md` ADR-004/005 + `risks-register.md` |
| GR / Public Affairs | `regulatory-roadmap.md` + `adr.md` ADR-004 + `operations-playbooks.md` (плейбук подключения партнёра) |
| HR / Recruiter, Финансист / CFO, Инвестор | работают только с NDA-материалами (`NDA/team-plan.md`, `NDA/tariff-model.md`, `NDA/financial-model.md`, `NDA/investment-plan.md`) — этот репозиторий им не нужен |
