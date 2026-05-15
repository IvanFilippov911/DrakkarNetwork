# Drakkar Network — доступ к приватной сети

## 1. Overview & Engineering Focus

Drakkar Network — это VPN-сервис с фокусом на стабильность, наблюдаемость и управляемость инфраструктуры. 

Главная идея: VPN-сервис должен не просто выдавать конфиг пользователю, а понимать состояние своей инфраструктуры и быстро реагировать на деградации.

### Operational focus

Проект сфокусирован на задачах, которые напрямую влияют на стабильность сервиса:

- контроль состояния VPN-серверов;
- сбор метрик, ошибок и алертов;
- детекция сетевых проблем;
- динамическое изменение сетевых профилей;
- оперативная миграция пользователей с проблемного сервера.

### Engineering focus

Витрина проекта показывает не только набор функций, но и то, как они реализованы внутри:

- модульная backend-архитектура;
- CQRS + MediatR для application layer;
- pipeline behaviors для validation, logging, metrics, Unit of Work и error events;
- async jobs для рискованных external side effects;
- node agents как изолированный слой управления VPN nodes;
- structured observability: ошибки, метрики, polling и operational events.

## 2. Структура репозитория

Репозиторий организован как витрина проекта Drakkar VPN: от общего описания системы до архитектуры, модулей, ключевых runtime-flow и решений по надёжности.

```text
DrakkarVpnShowcase/
├── README.md
├── docs/
│   ├── overview/        # продуктовый контекст, scope, ограничения
│   ├── architecture/    # C4, domain model, application pipeline
│   ├── flows/           # пошаговое описание ключевых runtime-flow
│   ├── modules/         # ответственность крупных модулей системы
│   └── decisions/       # ADR: архитектурные решения и trade-offs
├── diagrams/            # экспортированные схемы и исходники диаграмм
├── code-showcase/
│   └── DrakkarVpn/      # selected production-like code samples
├── tests/               # selected tests / architecture tests
└── deploy/              # reference configs / examples
```

### Как читать `docs/`

- **overview/** — зачем существует проект, какую проблему решает, какой scope и ограничения.
- **architecture/** — как система устроена на верхнем уровне: C4, domain model, application pipeline.
- **flows/** — как проходят ключевые сценарии во времени: provisioning, transport profile apply, polling, alerts.
- **modules/** — за что отвечают крупные части системы: Servers, Peers, Observability, Node Agent.
- **decisions/** — почему были выбраны конкретные архитектурные решения.

## 3. Recommended Review Path

Если вы смотрите репозиторий как backend/system design reviewer, лучше идти не по папкам подряд, а по основному инженерному сценарию.

### 1. Сначала понять систему сверху

- `docs/architecture/c4-system-context.md` — внешние пользователи, системы и границы Drakkar VPN.
- `docs/architecture/c4-container.md` — основные контейнеры: User API, Admin API, Node Agent, PostgreSQL, Redis, Observability stack, Xray runtime.
- `docs/architecture/domain-model.md` — ключевые сущности: User, Device, Peer, Server, TransportProfile, ServerTransportProfileActivation, ApplyJob.

### 2. Затем посмотреть главный showcase flow

- `docs/flows/server-transport-profile-apply.md` — как система применяет transport profile к VPN-серверу.
- `docs/decisions/ADR-003-async-apply-jobs.md` — почему применение transport profile выполняется асинхронно.

Этот flow показывает основную инженерную идею проекта: динамическое обновление сетевых профилей.

### 3. После этого открыть код

- `code-showcase/DrakkarVpn/modules/DrakkarVpn.Servers` — домен серверов, transport profiles, activations и apply jobs.
- `code-showcase/DrakkarVpn/hosts/DrakkarVpn.Admin.Api` — входная точка admin flow.
- `code-showcase/DrakkarVpn/hosts/DrakkarVpn.Agent` — node agent и интеграция с Xray runtime.

### 4. Потом посмотреть supporting engineering

- `docs/architecture/application-pipeline.md` — MediatR pipeline, validation, logging, metrics, Unit of Work, idempotency.
- `docs/modules/observability.md` — ошибки, метрики, алерты, polling серверов и operational visibility.

## 4. Code Showcase Scope

Этот репозиторий является витринной версией Drakkar VPN.

Он не предназначен для полноценного production-развёртывания и не является полной копией production-кода.

Основная цель `code-showcase/` — показать ключевые backend-механики, архитектурные решения и production-like подход к реализации.

### Что включено

- Server Transport Profile Apply flow;
- Peer Provisioning flow;
- Node Agent / Xray integration;
- Observability pipeline;
- MediatR / Unit of Work / Idempotency / Validation behaviors;
- selected domain entities and EF Core configurations;
- selected docs, diagrams and ADR.

### Вне фокуса витрины

Эти области сознательно не считаются главным сюжетом showcase:

- CRUD-only endpoints;
- Telegram Bot как продуктовая поверхность (не как инженерный сюжет витрины);
- полный production frontend (в витрине допускаются лишь выборочные фрагменты);
- billing-specific детали;
- hosting-specific инфраструктура;
- временные эксперименты и внутренний рабочий шум.

### Почему так

Цель репозитория — не запустить полный VPN-сервис локально, а показать самые важные инженерные части проекта.

## 5. Tech Stack

### Backend

- C# / .NET 8
- ASP.NET Core: REST APIs, background services
- MediatR + FluentValidation
- EF Core + PostgreSQL / Npgsql
- Redis
- Serilog
- prometheus-net
- Swagger / OpenAPI

### Architecture & Application Layer

- Modular Monolith с разбиением по bounded contexts
- CQRS-style commands/queries через MediatR
- Pipeline behaviors: validation, logging, metrics, Unit of Work, idempotency, error events / alerting
- Background workers и async job processing: lease, retry, permanent failure, obsolete jobs

### VPN Runtime & Control Plane

- Xray core
- VLESS + REALITY
- Node Agent как отдельный сервис для управления VPN nodes
- HTTP API между control plane и node agents
- Интеграция agent'а с Xray local API / gRPC

### Observability

- structured logs
- Prometheus metrics
- доменные error events
- operational signals / alerts
- server polling
- reference stack: Prometheus, Grafana, Loki, Promtail

### Infrastructure

- Docker / Docker Compose
- Nginx
- EF Core migrations / DbMigrator

### Frontend

- React
- Vite
- TypeScript

## 6. Project Status

Продукт **Drakkar VPN** находится в стадии рабочего MVP.

Уже реализованы и задеплоены в базовом production-сценарии: пользовательский flow получения доступа, админская часть, backend API (user/admin), управление пользователями, устройствами и VPN peer'ами, node agent, интеграция с Xray runtime, а также базовая наблюдаемость (логи, метрики, ошибки, polling состояния серверов).

В активной разработке:
- автоматическая детекция сетевых деградаций;
- автоматическая миграция пользователей с проблемных серверов.
