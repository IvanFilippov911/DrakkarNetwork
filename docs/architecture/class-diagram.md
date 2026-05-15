# Domain Model

Документ показывает ключевые доменные сущности Drakkar Network и связи между ними.

![Domain Model](./assets/class-diagram.svg)

## Назначение диаграммы

Диаграмма нужна, чтобы быстро понять основную модель данных и доменные связи.
Это не полная ERD всей системы, а архитектурная domain model для основных showcase-flow.

## Основные зоны модели

### User Access

Зона отвечает за пользователя, его устройства и доступ к сервису.

Ключевые сущности:

- `AppUser` — пользователь сервиса, связанный с Telegram account;
- `Device` — устройство пользователя, для которого может быть выдан VPN-доступ;
- `Subscription` — активный доступ пользователя к сервису;
- `Tariff` — тарифные условия, из которых формируются параметры подписки.

Основные связи:

- один `AppUser` может иметь несколько `Device`;
- у пользователя может быть одна активная `Subscription`;
- `Tariff` логически связан с `Subscription` как источник условий при создании подписки.

`Subscription` хранит фактические условия доступа пользователя, например `MaxDevices` и `Status`.

Связь с `Tariff` показана как logical, потому что тариф используется как источник условий при создании подписки, а не как обязательная runtime-зависимость для каждого сценария.

### VPN Access

Зона отвечает за VPN peer'ы и процесс их выдачи.

Ключевые сущности:

- `Peer` — VPN identity, выданная конкретному устройству на конкретном сервере;
- `PeerProvisionJob` — асинхронная операция создания peer'а через infrastructure flow.

Основные связи:

- одно `Device` может иметь несколько `Peer`;
- один `Server` может обслуживать несколько `Peer`;
- `PeerProvisionJob` создаётся для конкретного `Device` и целевого `Server`.

`Peer` связывает устройство пользователя с конкретным VPN-сервером.

`PeerProvisionJob` нужен, потому что создание peer'а связано с external side effect: нужно применить изменение на VPN node через agent / runtime.

### Infrastructure Control

Зона отвечает за серверы и transport profiles.

Ключевые сущности:

- `Server` — VPN node с точки зрения control plane;
- `TransportProfile` — шаблон сетевого профиля;
- `ServerTransportProfileActivation` — привязка transport profile к конкретному серверу;
- `ServerTransportApplyJob` — асинхронное применение выбранного transport profile к серверу.

Основные связи:

- один `Server` может иметь несколько `ServerTransportProfileActivation`;
- один `TransportProfile` может быть привязан к нескольким серверам через activations;
- `ServerTransportApplyJob` создаётся для конкретного `Server`, `ActivationId` и `TargetTransportVersion`.

`Server` хранит desired transport state:

- `DesiredTransportActivationId`;
- `DesiredTransportVersion`.

Это позволяет системе отличать желаемое состояние от фактически применённого состояния на node.

## Ключевые инварианты

Инварианты ниже — правила доменной модели, которые поддерживают корректность showcase-flow. Они не претендуют на полноту всей ERD.

### User / Device / Subscription

- один `AppUser` владеет многими `Device`, каждое `Device` принадлежит ровно одному пользователю;
- право на выдачу VPN-доступа опирается на **активную** `Subscription` и её лимиты (например `MaxDevices`, `Status`);
- условия подписки после создания хранятся на `Subscription`; `Tariff` не обязан сопровождать каждый runtime-сценарий.

### Peer и provisioning

- `Peer` всегда привязан к конкретной паре **устройство + сервер** (идентичность доступа в разрезе node);
- на одном `Server` может существовать много `Peer` (по разным устройствам);
- выдача peer'а через `PeerProvisionJob` — **асинхронная** операция с внешним side effect (node agent / runtime), а не синхронный CRUD.

### Transport profiles и activations

- связь **сервер ↔ профиль** моделируется через `ServerTransportProfileActivation`; один `TransportProfile` может использоваться на нескольких серверах;
- у одного `Server` может быть несколько activations (история / подготовленные профили), но **в каждый момент времени не более одной активной** activation, задающей фактический transport для node;
- **активную** activation нельзя безопасно «просто удалить»: сначала нужно перевести сервер на другой допустимый профиль (смена desired / активации), иначе ломается согласованность control plane и data plane.

### Apply jobs (transport)

- `ServerTransportApplyJob` фиксирует **попытку** довести node до состояния, заданного `ActivationId` и `TargetTransportVersion`;
- job ссылается на `ServerId`, `ActivationId` и целевую версию; обработчик опирается на lease и идемпотентные переходы состояния;
- если **desired** transport на `Server` изменился (другая версия / другая цель), неактуальная job может быть помечена как **obsolete** — повторно гонять старую попытку нельзя;
- **успешное применение** на node и **фиксация applied state** в control plane — отдельный шаг от смены **desired** state: desired может обновиться раньше, чем node догонит.

## Связь с showcase-flow

Диаграмма помогает понять два основных flow, которые дальше разобраны в документации:

- `ServerTransportApplyJob` показывает async-flow применения transport profile к VPN-серверу.
- `PeerProvisionJob` показывает async-flow выдачи VPN peer'а для устройства пользователя.

Обе операции вынесены в job-модель, потому что затрагивают внешний runtime через node agent и не должны выполняться как простой синхронный CRUD.

