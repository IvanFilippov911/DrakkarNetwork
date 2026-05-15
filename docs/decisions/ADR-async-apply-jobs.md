# ADR— Async Apply Jobs для применения transport profile

## Status

Accepted

## Context

Смена `transport profile` для VPN-сервера затрагивает не только состояние в базе данных, но и внешний runtime на VPN node.

После выбора нового профиля системе нужно применить изменение на сервере через `VPN Node Agent`, который взаимодействует с `Xray Runtime`.

Эта операция является внешним side effect и может завершиться нестабильно:

- node agent может быть недоступен;
- Xray runtime может вернуть ошибку;
- сетевой вызов может завершиться timeout'ом;
- команда может стать устаревшей, если администратор выбрал другой profile;
- результат применения должен быть виден администратору.

Если выполнять эту операцию напрямую внутри HTTP request, Admin API станет зависеть от доступности node agent и Xray runtime. Это ухудшает надёжность, усложняет retry и делает результат операции плохо наблюдаемым.

## Decision

Применение `transport profile` выполняется через асинхронную job-модель.

Admin API не вызывает node agent напрямую как часть основного HTTP request.

Вместо этого система:

1. сохраняет новое desired state сервера;
2. увеличивает desired transport version;
3. создаёт `ServerTransportApplyJob`;
4. фиксирует эти изменения в одной транзакции;
5. background worker позже забирает job;
6. worker вызывает `VPN Node Agent`;
7. результат применения фиксируется в статусе job.

## Consequences

### Positive

- HTTP request остаётся быстрым и не зависит напрямую от node runtime.
- Изменение desired state и создание apply job фиксируются атомарно.
- Ошибки применения становятся видимыми через состояние job.
- Можно реализовать retry для временных ошибок.
- Можно помечать устаревшие jobs как `Obsolete`.
- Worker может обрабатывать jobs независимо от Admin API.
- Администратор получает более прозрачное состояние операции.

### Negative

- Система становится eventually consistent.
- Между изменением desired state и фактическим применением на node появляется задержка.
- Требуется отдельная модель job lifecycle.
- Нужно поддерживать worker, lease/guard и обработку failed/retry/obsolete состояний.
- Админский UI должен корректно показывать промежуточные состояния.

## Job Lifecycle

`ServerTransportApplyJob` проходит через несколько основных состояний:

- `Pending` — job создана и ожидает обработки;
- `Processing` — job взята worker'ом;
- `Completed` — profile успешно применён;
- `Failed` — постоянная ошибка или превышен лимит попыток;
- `Obsolete` — job устарела из-за нового desired state.

## Obsolete Jobs

Job считается устаревшей, если её `TargetTransportVersion` больше не совпадает с актуальной desired version сервера.

Это защищает систему от ситуации, когда старая команда применяется после того, как администратор уже выбрал новый transport profile.

## Result

Для применения `transport profile` используется async apply job.

Это усложняет реализацию, но делает flow более надёжным, наблюдаемым и управляемым.

Связанный flow:

- `docs/flows/server-transport-profile-apply.md`







