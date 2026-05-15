# Server Transport Profile Apply Flow

Документ описывает flow применения `transport profile` к VPN-серверу.

Это один из ключевых control-plane сценариев Drakkar Network: администратор меняет сетевой профиль сервера, а система безопасно применяет это изменение через async job и node agent.

![Server Transport Profile Apply Sequence](./assets/server-transport-profile-apply-sequence.jpeg)

## Назначение flow

Flow описывает применение `transport profile` к VPN-серверу.

Сценарий начинается с выбора profile activation в Admin API и заканчивается фиксацией результата `ServerTransportApplyJob`.

## Участники

### Admin WebApp

Интерфейс администратора.

Через него администратор выбирает новый `transport profile` для сервера.

### Admin API

Принимает команду на активацию профиля.

Отвечает за:

- валидацию запроса;
- изменение desired state сервера;
- создание `ServerTransportApplyJob`;
- сохранение изменений в PostgreSQL.

### PostgreSQL

Хранит состояние control plane:

- серверы;
- transport profiles;
- activations;
- desired transport state;
- apply jobs;
- статусы выполнения.

### Background Worker

Обрабатывает pending apply jobs.

Отвечает за:

- получение job из очереди/БД;
- проверку актуальности job;
- вызов node agent;
- фиксацию результата.

### VPN Node Agent

Сервис на VPN node.

Принимает команду от control plane и применяет новый transport profile к локальному Xray runtime.

### Xray Runtime

VPN runtime, который обслуживает реальные подключения пользователей.

## Основной сценарий

1. Администратор выбирает новый `transport profile` для сервера.
2. Admin WebApp отправляет запрос в Admin API.
3. Admin API проверяет, что сервер и activation существуют.
4. Система обновляет desired transport state сервера.
5. В той же транзакции создаётся `ServerTransportApplyJob`.
6. Background Worker забирает pending job.
7. Worker проверяет, что job всё ещё актуальна.
8. Worker вызывает VPN Node Agent.
9. Node Agent применяет изменения к Xray Runtime.
10. Worker фиксирует результат job в PostgreSQL.

## State Machine

Жизненный цикл apply job показан на state machine diagram.

![Server Transport Profile Apply State Machine](./assets/server-transport-profile-apply-state-machine.jpeg)

## Основные состояния job

### Pending

Job создана и ожидает обработки.

### Processing

Job взята worker'ом в обработку.

На этом этапе система применяет lease/guard, чтобы несколько worker'ов не обработали одну job одновременно.

### Completed

Transport profile успешно применён на node.

### Failed

Job завершилась ошибкой, которую нельзя безопасно повторять автоматически.

### Retry / Scheduled

Временная ошибка: agent недоступен, timeout, временная проблема сети или runtime.

Job может быть повторена позже.

### Obsolete

Job устарела.

Это происходит, если desired state сервера изменился после создания job. Например, администратор выбрал другой transport profile до того, как старая job была применена.


## Code References

