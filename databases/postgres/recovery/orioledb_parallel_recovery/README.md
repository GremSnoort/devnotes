# OrioleDB Parallel Recovery: full model, finalization and visibility

Этот документ описывает полную модель `parallel recovery` в OrioleDB:

- роли startup/main recovery process и recovery worker-ов;
- какие shared watermark-и используются для координации;
- какие существуют пути финализации transaction state;
- где именно происходит deferred finalization;
- где публикуется глобальная transaction visibility.

Описание опирается на код OrioleDB из:

- `src/recovery/recovery.c`
- `src/recovery/worker.c`
- `src/orioledb.c`

## Общая модель parallel recovery в OrioleDB

Во время recovery OrioleDB использует:

- **startup/main recovery process**;
- **параллельные recovery worker-ы**;
- в отдельных сценариях также специальный **index-build worker subpool**, управляемый через `recovery_idx_pool_size_guc` и `index_build_leader`; это отдельный специализированный путь для parallel index build during recovery.

Роли распределены так:

- **startup/main process** читает WAL, парсит контейнеры и записи, координирует replay, рассылает часть работы worker-ам, продвигает глобальные recovery pointers и выполняет финальную публикацию global transaction visibility;
- **worker-ы** применяют свою часть redo, поддерживают локальное transaction state и продвигают собственный replay progress;
- **финализация visibility** в общем случае не делается worker-ами напрямую, а проходит через deferred model с локальным состоянием recovery process-ов и глобальными watermark-ами.

Концептуально Oriole recovery **не придерживается** модели `каждый commit = полный глобальный barrier`.

В Oriole используется асинхронная модель:

- main process координирует replay;
- worker-ы доигрывают свою часть независимо;
- глобальная видимость части транзакций публикуется позже, когда для конкретного recovery process достигнута его publication-safe boundary: для startup/main process это `recovery_get_current_ptr()`, а для worker-а — уже опубликованная global boundary в `recovery_finished_list_ptr`.

В Oriole важно различать:

- **локальное завершение transaction path** в конкретном recovery process;
- **глобальную финализацию transaction state**;
- **публикацию visibility** для transaction semantics.

## Какие механизмы финализации transaction state есть в Oriole recovery

В Oriole recovery финализация transaction state использует несколько взаимосвязанных путей, которые отвечают за разные стадии завершения и публикации состояния транзакции.

Основные механизмы:

- **локальный finish через `recovery_finish_current_oxid()`**
  Этот шаг завершает transaction path в конкретном recovery process. В зависимости от режима (`sync = true` или `sync = false`) он либо сразу доводит transaction state до финального состояния, либо переводит его в deferred finalization.

- **`finished_list`**
  Это механизм отложенной финализации. В него попадают transaction states, которые уже завершены локально, но еще не доведены до globally published state. Дальнейшая публикация происходит позже, когда startup/main process догоняет соответствующую boundary.

- **`joint_commit_list`**
  Это отдельный путь для транзакций, чья финализация привязана к builtin PostgreSQL commit/abort path. Такие транзакции сначала ждут соответствующего builtin commit/abort события и только потом переходят в обычную finish/finalization logic.

- **`update_proc_retain_undo_location()`**
  Это центральный механизм, который связывает локально завершенные transaction states с глобальной publication boundary. Для `finished_list`-based deferred-finalization path именно здесь startup/main process публикует финальный CSN / XLOG pointer и продвигает `recovery_finished_list_ptr`.

- **shared watermark-и (`recovery_ptr`, `commitPtr`, `recovery_finished_list_ptr`, `retainPtr`)**  
  Эти указатели задают границы, относительно которых recovery process-ы понимают:
  - до какого LSN уже продвинут replay;
  - до какого LSN worker завершил локальную обработку;
  - до какого LSN startup уже выполнил глобальную publication;
  - до какого LSN продвинуто retain/undo bookkeeping.

Полная модель финализации в Oriole recovery состоит из нескольких уровней:

- локальное завершение transaction path;
- возможное ожидание builtin joint commit;
- deferred finalization через `finished_list`;
- финальная публикация global transaction visibility через startup/main process.

Это делает recovery pipeline асинхронным, но при этом позволяет отделить:

- физическое применение redo;
- локальное завершение transaction state;
- и момент, когда состояние становится globally visible.

## Per-process state и shared state

В recovery коде есть четкое различие между:

- **локальным состоянием recovery process-а**;
- **shared watermark-ами**, доступными всем recovery process-ам.

### Локальное состояние recovery process-а

У каждого recovery process-а есть собственные локальные структуры:

- `recovery_xid_state_hash` — локальный hash transaction state для этого process-а;
- `finished_list` — локальный список транзакций, завершенных локально, но еще не доведенных до окончательной global publication;
- `joint_commit_list` — локальный список транзакций, ожидающих joint commit с builtin PostgreSQL transaction;
- локальные retain/xmin queues.

Это видно в `recovery_init()`:

- инициализируется локальный `recovery_xid_state_hash`;
- отдельно `dlist_init(&finished_list)`;
- отдельно `dlist_init(&joint_commit_list)`.

`finished_list` — это **локальный список** конкретного recovery process-а.

### Shared watermark-и

Для координации между process-ами используются общие shared pointer-ы:

- `recovery_ptr` — глобальный watermark replay/dispatched progress, который поддерживает startup/main process: до какого LSN работа уже продвинута и/или разослана дальше по recovery pipeline;
- `worker_ptrs[i].commitPtr` — replay progress watermark конкретного worker-а;
- `recovery_finished_list_ptr` — глобальный watermark финальной publication boundary: до какого LSN startup/main process уже догнал deferred finalization и довел transaction visibility до globally published state;
- `worker_ptrs[i].retainPtr` / `recovery_main_retain_ptr` — watermark-и retain/undo bookkeeping.

Ключевое различие между `recovery_ptr` и `recovery_finished_list_ptr`:

- `recovery_ptr` — это **watermark replay/dispatched progress**, который продвигает startup/main process по мере того, как работа раздается и движется дальше по WAL/replay pipeline;
- `recovery_finished_list_ptr` — это **watermark финальной publication boundary**: до какого LSN startup/main process уже догнал deferred finalization и опубликовал global transaction visibility.

Иными словами:

- `recovery_ptr` отвечает на вопрос: **до какого LSN startup уже продвинул/разослал recovery pipeline**;
- `recovery_finished_list_ptr` отвечает на вопрос: **до какого LSN deferred finalization уже доведена до globally visible state**.
- `recovery_get_current_ptr()` в parallel mode отвечает уже на другой вопрос: **до какого LSN все recovery worker-ы реально дошли**, то есть какая boundary является safe fully-reached boundary для startup/main process.

Reader / writer semantics:

- `recovery_ptr`
  - **writer**: startup/main process, который продвигает общую границу replay;
  - **reader**: startup/main process, recovery worker-ы и code paths синхронизации, которым нужно понимать текущую глобальную границу WAL/replay progress.

- `recovery_finished_list_ptr`
  - **writer**: только startup/main process, который продвигает его при обработке `finished_list` в `update_proc_retain_undo_location()`;
  - **reader**: startup/main process и recovery worker-ы, которые используют этот pointer как already-published global boundary для локальной deferred-finalization logic и retire local state.

Дополнительно:

- `commitPtr` означает: worker уже продвинул свой локальный replay progress до этого LSN;
- `recovery_finished_list_ptr` означает именно watermark уже опубликованной global visibility boundary.

## Как создается и ведется transaction state

Ключевой объект здесь — `RecoveryXidState`.

В нем хранятся, в частности:

- `oxid`
- `xid`
- `csn`
- `ptr`
- `in_finished_list`
- `in_joint_commit_list`
- `needs_feedback`
- `systree_modified`
- `checkpoint_xid`
- `wal_xid`
- `used_by`

`RecoveryXidState` живет в локальном `recovery_xid_state_hash` конкретного recovery process-а.

Когда process переключается на новый `OXid` через `recovery_switch_to_oxid()`:

- ищется или создается соответствующий `RecoveryXidState`;
- локальное undo/checkpoint состояние привязывается к нему;
- при необходимости обновляется retain bookkeeping;
- вызывается `update_proc_retain_undo_location(worker_id)`.

Finalization bookkeeping встроен в обычную эволюцию recovery transaction state.

## Что означает recovery_finish_current_oxid()

`recovery_finish_current_oxid()` — это ключевая функция **локального завершения transaction path** для текущего recovery process-а.

Она вызывается:

- startup/main process-ом;
- worker-ами;
- для commit и rollback.

У нее есть два принципиально разных режима:

- `sync = true`
- `sync = false`

### sync = true

При `sync = true` функция делает полную финализацию сразу:

- выставляет `COMMITSEQNO_COMMITTING`;
- выполняет `precommit_undo_stack()` / `on_commit_undo_stack()`;
- проходит `walk_checkpoint_stacks()`;
- выделяет финальный CSN;
- записывает `set_oxid_csn(...)`;
- записывает `set_oxid_xlog_ptr(...)`.

Transaction state становится финальным и globally published непосредственно в момент вызова.

### sync = false

При `sync = false` функция делает только локальную фазу finish.

Для commit path:

- выполняет precommit/on-commit действия;
- проходит checkpoint stacks;
- **не назначает финальный CSN и не публикует xlog ptr как окончательный**;
- кладет state в локальный `finished_list`.

Для abort path важно различать два случая:

- **startup/main process** может отложить abort через `finished_list`;
- **worker** при rollback и `sync = false` применяет undo локально и не кладет rollback в `finished_list`.

`recovery_finish_current_oxid(..., sync = false)` не означает, что transaction уже fully visible, а показывает, что:

- локальная часть завершена,
- state переведен в **deferred finalization**.

## Finished-list path: deferred finalization

`finished_list` — это **локальный список** recovery process-а, содержащий транзакции, которые:

- уже прошли локальную фазу finish;
- но еще не были окончательно доведены до globally published state.

Для `RecoveryXidState` в этом списке особенно важны:

- `state->csn`
- `state->ptr`
- `state->in_finished_list`

Пока state находится в `finished_list`, final publication еще не завершена.

Это основная deferred-finalization ветка Oriole recovery.

## Joint-commit path: не все транзакции сразу попадают в finished_list

Отдельный путь финализации — `joint_commit_list`.

Он нужен для транзакций, чья финализация привязана к builtin PostgreSQL commit/abort path.

Это видно в `o_xact_redo_hook()`:

- функция итерирует `joint_commit_list`;
- находит state с matching builtin `xid`;
- при необходимости шлет worker-ам `workers_send_oxid_finish(...)`;
- иногда делает `workers_synchronize(...)` для специальных случаев;
- удаляет state из `joint_commit_list`;
- затем вызывает `recovery_finish_current_oxid(...)`.

Важный вывод:

- не каждая transaction finalization изначально стартует через `finished_list`;
- часть транзакций сначала проходит через `joint_commit_list` и только потом попадает в обычную finish/finalization logic.

## Как worker-ы завершают свою часть

В `worker.c` при получении `RecoveryMsgTypeCommit` и `RecoveryMsgTypeRollback` worker делает:

1. `recovery_switch_to_oxid(...)`
2. `recovery_finish_current_oxid(..., sync = false)`
3. `update_worker_ptr(id, ptr)`

То есть worker сообщает два факта:

- его локальная transaction path завершена;
- его локальный replay progress продвинут до `ptr`.

`commitPtr` используется шире, чем только как completion pointer commit/rollback. Он также обновляется, например:

- на `RecoveryMsgTypeSynchronize`;
- на `RecoveryMsgTypeRollbackToSavepoint`;
- в fast-path сценариях пустой очереди.

`commitPtr` корректнее понимать как общий watermark локального replay progress worker-а.

## Как startup узнает, что нужно догнать finalization

В `recovery_queue_read()` worker может разбудить startup через `WakeupRecovery()`.

Это происходит, если:

- нужен feedback primary;
- или worker уже догнал `recovery_ptr`, но `recovery_finished_list_ptr` еще отстает.

Комментарий в коде:
```txt
CSN assignment and finishedPtr update are pending
```

Worker понимает:

- локальный replay progress уже достаточный;
- но startup еще не продвинул global publication boundary.

## Где происходит финальная publication boundary: update_proc_retain_undo_location()

`update_proc_retain_undo_location()` — это центральная функция, где intersect-ятся:

- retain undo bookkeeping;
- local deferred-finalization state;
- и global publication watermark.

Важно различать общую и main-only части.

### Общая логика, возможная в любом recovery process

Функция может:

- обновлять retain undo bookkeeping;
- обрабатывать локальный `finished_list` до допустимой границы;
- обновлять retain pointers.

### Логика только для startup/main process

Только startup/main process:

- использует `listPtr = recovery_get_current_ptr()`;
- назначает финальный CSN / final xlog ptr;
- продвигает `recovery_finished_list_ptr`.

Worker-ы, напротив:

- используют `recovery_finished_list_ptr` как входную global boundary;
- догоняют только свой локальный `finished_list` до уже опубликованной границы;
- сами global watermark не продвигают.

### Как выбирается boundary

Внутри функции:

- если `worker_id < 0`, то `listPtr = recovery_get_current_ptr()`;
- иначе `listPtr = recovery_finished_list_ptr`.

Это означает:

- startup/main process имеет право догонять свой локальный `finished_list` до текущего общего recovery progress;
- worker может догонять свой локальный `finished_list` только до той границы, которую уже опубликовал startup.

### Что именно делает startup при final publication

Когда startup/main process проходит по локальному `finished_list`, для каждого `state` с `state->ptr <= listPtr` он:

- для commit path:
  - выставляет `COMMITSEQNO_COMMITTING`;
  - выделяет финальный CSN;
  - записывает `set_oxid_csn(...)`;
  - записывает `set_oxid_xlog_ptr(...)`;
- для abort path:
  - выставляет `COMMITSEQNO_ABORTED`;
  - выставляет `InvalidXLogRecPtr`.

После этого:

- state удаляется из `finished_list`;
- `state->in_finished_list = false`;
- вызывается `check_delete_xid_state(...)`.

Здесь transaction state перестает быть deferred и становится globally published.

## Что означает recovery_finished_list_ptr

После обхода `finished_list` startup/main process делает:

- `pg_atomic_write_u64(recovery_finished_list_ptr, recoveryPtr);`

Это означает:

- до этого LSN global publication boundary уже продвинута;
- startup завершил final publication для тех локальных states, которые мог финализировать на своей стороне до этой точки.

`recovery_finished_list_ptr` — это **глобальный watermark финальной публикации**.

## Где именно публикуется global transaction visibility

Важно различать два слоя:

- **физическое применение redo** worker-ами, которое происходит раньше;
- **финальную публикацию global transaction state**, которая происходит позже.

Финальная публикация global transaction visibility происходит тогда, когда startup/main process в `update_proc_retain_undo_location()`:

- назначает финальный CSN;
- выставляет final `set_oxid_xlog_ptr(...)`;
- удаляет state из локального `finished_list`;
- продвигает `recovery_finished_list_ptr`.

Локальные `finished_list` recovery process-ов вместе с `recovery_finished_list_ptr` образуют буфер между:

- асинхронной replay-работой worker-ов;
- и моментом, когда transaction state становится fully published с точки зрения Oriole transaction visibility semantics.

## Почему это важно для targeted recovery

Эта модель важна для recovery target semantics.

PostgreSQL может решить, что target point reached, раньше, чем Oriole startup process доведет свою final-publication boundary до той же точки.

Тогда возможно состояние, когда:

- PostgreSQL уже считает, что recovery остановилась в нужной boundary;
- worker-ы уже физически применили значимую часть redo;
- часть transaction path уже локально завершена;
- но Oriole-visible state еще не fully published, потому что startup еще не догнал deferred finalization до этой boundary.

Это различие между:

- `target reached` со стороны PostgreSQL;
- и `global transaction visibility fully published` со стороны Oriole

## Вывод

Полная модель `parallel recovery` в OrioleDB состоит из нескольких уровней:

- startup/main process координирует replay и управляет global publication boundary;
- worker-ы независимо доигрывают свою часть redo и продвигают собственный replay progress;
- часть транзакций идет через `joint_commit_list`;
- часть transaction state попадает в deferred-finalization path через локальный `finished_list`;
- `update_proc_retain_undo_location()` соединяет local deferred state с global publication semantics;
- `recovery_finished_list_ptr` показывает, до какого LSN global final publication уже продвинута.

Поэтому при анализе visibility проблем в Oriole recovery важно различать как минимум четыре разных состояния:

- `worker replay progressed`;
- `local transaction path finished`;
- `startup caught up with deferred finalization`;
- `global transaction visibility published`.
