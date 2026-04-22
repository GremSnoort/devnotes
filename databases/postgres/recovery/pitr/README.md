# PITR

`PITR` (`Point-In-Time Recovery`) в PostgreSQL - это конкретный режим `archive recovery`, в котором startup-процесс:

- поднимается из `base backup` или другого восстановленного состояния кластера;
- читает WAL из `pg_wal` и/или архива;
- применяет записи WAL последовательно;
- останавливается в **заранее заданной целевой точке** (а не в самом конце доступного WAL).

Технически это поведение размазано в основном по:

- `src/backend/access/transam/xlog.c`
- `src/backend/access/transam/xlogrecovery.c`
- `src/backend/access/transam/xlogfuncs.c`
- `src/backend/utils/misc/guc_tables.c`

Ниже описание опирается именно на этот код.

## GUC-переменные

- `archive_command` — как PostgreSQL складывает WAL наружу. Обычно используется на узле, который архивирует WAL.
- `restore_command` — как PostgreSQL забирает WAL обратно из архива во время recovery.
- `primary_conninfo` — как standby подключается к живому primary и получает WAL по сети.

`archive_command` и `restore_command` обычно образуют пару для backup/recovery/PITR.

`primary_conninfo` нужен для standby и streaming replication.

Standby может использовать и `restore_command`, и `primary_conninfo` одновременно:
- сначала брать WAL из архива;
- потом догонять primary по стриму.

## Что именно считается PITR в коде

В текущей реализации PostgreSQL полезно различать три близких режима:

- `crash recovery` - сервер просто поднимается после некорректного завершения и докатывает локально доступный WAL;
- `archive recovery` - сервер вошел в recovery по `recovery.signal` или `standby.signal` и может тянуть WAL из архива или от primary;
- `PITR` / `targeted recovery` - частный случай `archive recovery`, где задана одна из GUC-переменных `recovery_target*`, и replay должен остановиться в целевой точке.

Это хорошо видно в `InitWalRecovery()` в `xlogrecovery.c`:

- наличие `recovery.signal` или `standby.signal` выставляет `ArchiveRecoveryRequested = true`;
- если archive recovery запрошен **не в standby mode** и заданы GUC-переменные `recovery_target_time`, `recovery_target_xid`, `recovery_target_name`, `recovery_target_lsn` или `recovery_target = 'immediate'`, startup логирует именно `starting point-in-time recovery ...`;
- если archive recovery запрошен **не в standby mode** и recovery target не указан, логируется `starting archive recovery`.

По коду, `PITR` - это именно **archive recovery с точкой остановки**.

### Что такое crash recovery

`crash recovery` в PostgreSQL - это режим startup recovery, в котором сервер пытается вернуть кластер в согласованное состояние после некорректного завершения, используя WAL, уже доступный локально в `pg_wal`.

По коду это ветка общего recovery pipeline. В `InitWalRecovery()` решение о входе в recovery принимается по состоянию checkpoint и `pg_control`:

- если `checkPoint.redo < CheckPointLoc`, recovery обязателен;
- если `ControlFile->state != DB_SHUTDOWNED`, recovery тоже обязателен;
- если сервер был остановлен чисто, но присутствует `recovery.signal` или `standby.signal`, recovery принудительно включается уже как **archive recovery**.

`crash recovery` - это восстановление **после аварийного состояния кластера** (не после явного DBA-запроса на restore).

Ключевой признак:

- `InRecovery = true`
- `ArchiveRecoveryRequested = false` или еще не перешел в активную archive phase
- `ControlFile->state = DB_IN_CRASH_RECOVERY`

Это видно в `InitWalRecovery()` в `xlogrecovery.c`: если recovery нужен, но `InArchiveRecovery` еще не выставлен, startup пишет в лог `database system was not properly shut down; automatic recovery in progress` и помечает cluster state как `DB_IN_CRASH_RECOVERY`.

С практической точки зрения `crash recovery` означает:

- старт от последнего валидного checkpoint;
- чтение WAL из локального `pg_wal`;
- применение WAL до достижения консистентного состояния, а обычно и до конца доступного локального WAL;
- отсутствие recovery target semantics как бизнес-точки остановки.

`crash recovery` и `archive recovery` могут идти подряд в рамках одного запуска.

В `xlogrecovery.c` есть прямой комментарий: если `ArchiveRecoveryRequested = true`, но `InArchiveRecovery = false`, startup пока делает recovery только по WAL из `pg_wal`, а в archive mode переключится позже, когда дойдет до конца локального WAL.

### Что такое archive recovery

`archive recovery` в PostgreSQL - это режим recovery, в котором startup-процесс восстанавливает кластер в контексте запрошенного archive recovery: replay может начаться с локального `pg_wal`, а затем при необходимости продолжиться WAL из архива через GUC-переменную `restore_command` или, в standby-сценарии, от primary.

`restore_command` — это GUC-переменная PostgreSQL, в которой задается shell-команда для получения нужного WAL-файла из архива во время recovery. С помощью `restore_command` PostgreSQL во время recovery извлекает недостающие WAL-файлы из внешнего архива, чтобы продолжить replay.

Replay может продолжиться WAL из архива через `restore_command`:

- сначала recovery может читать WAL из локального `pg_wal`;
- когда локально нужного сегмента уже нет, но recovery нужно идти дальше;
- PostgreSQL начинает пытаться восстанавливать следующие WAL-файлы из архива через `restore_command`;
- если команда успешно отдает файл, replay продолжается.

То есть `restore_command` — это мост между PostgreSQL recovery и внешним архивом WAL.

В коде здесь важно различать два близких флага из `xlogrecovery.c`:

- `ArchiveRecoveryRequested` - archive recovery был запрошен, то есть при старте были обнаружены `recovery.signal` или `standby.signal`;
- `InArchiveRecovery` - startup уже находится в фазе recovery, где реально использует offline archive / archive-oriented sourcing WAL.

Это distinction прямо зафиксировано в комментарии в `xlogrecovery.c`:

- если `ArchiveRecoveryRequested = true`, но `InArchiveRecovery = false`, сервер еще может временно делать **crash recovery** по WAL из `pg_wal`;
- когда локальный WAL заканчивается, startup переключается в archive phase и выставляет `InArchiveRecovery = true`.

Вход в archive recovery начинается через `readRecoverySignalFile()`:

- `recovery.signal` включает archive recovery;
- `standby.signal` включает и archive recovery, и standby mode.

Дальше `validateRecoveryParameters()` проверяет, что такой режим вообще допустим:

- для **non-standby archive recovery** обязательна GUC-переменная `restore_command`;
- для **standby archive recovery** допускается также **поток WAL с primary**;
- затем валидируются GUC-переменные `recovery_target_*` и `recovery_target_timeline`.

С точки зрения состояния кластера **archive recovery** в коде маркируется как:

- `ArchiveRecoveryRequested = true`
- `InArchiveRecovery = true`
- `ControlFile->state = DB_IN_ARCHIVE_RECOVERY`

В operational terms `archive recovery` означает:

- сервер поднят в специальном recovery-режиме;
- source of truth для движения вперед - это WAL stream после checkpoint/base backup;
- replay WAL может идти:
    - сначала из `pg_wal`,
    - затем при необходимости из архива,
    - а в standby mode также от upstream;
- recovery может идти:
    - либо до конца доступного WAL,
    - либо до **configured recovery target**.

`archive recovery` - более широкое понятие, чем `PITR`.

Если сервер просто стартовал по `recovery.signal` и докатился до конца доступного WAL, это **archive recovery**.
Если в этом же режиме задана одна из GUC-переменных `recovery_target_time`, `recovery_target_xid`, `recovery_target_name`, `recovery_target_lsn` или `recovery_target = 'immediate'`, это уже `PITR` как **частный случай archive recovery**.

## С чего PITR начинается

Вход в recovery начинается в `StartupXLOG()` в `src/backend/access/transam/xlog.c`.

Общий порядок такой:

1. Проверяется состояние кластера по `pg_control`.
2. Приводится в порядок `pg_wal` и структура каталогов.
3. Вызывается `InitWalRecovery(ControlFile, ...)`.
4. После инициализации shared state запускается replay через `PerformWalRecovery()`.
5. После завершения replay выполняется `FinishWalRecovery()`, затем end-of-recovery действия.

Ключевая часть - `InitWalRecovery()`:

- читает signal-файлы через `readRecoverySignalFile()`;
- валидирует recovery-параметры через `validateRecoveryParameters()`;
- читает `backup_label`, если он есть;
- выбирает checkpoint и redo start;
- определяет, нужен ли `crash recovery`, `archive recovery` или их комбинация;
- инициализирует `InRecovery`, `InArchiveRecovery`, `ArchiveRecoveryRequested`, `StandbyModeRequested`.

## Как сервер понимает, что надо делать PITR

### Signal files

Функция `readRecoverySignalFile()` в `xlogrecovery.c`:

- не поддерживает старый `recovery.conf` и аварийно падает, если файл найден;
- проверяет `standby.signal` и `recovery.signal`;
- если найден `standby.signal`, выставляет и `StandbyModeRequested = true`, и `ArchiveRecoveryRequested = true`;
- если найден `recovery.signal`, выставляет `ArchiveRecoveryRequested = true`, но не standby mode.

**PITR в PostgreSQL включается не отдельным конфигом, а signal-файлом плюс recovery GUC-параметрами.**

### Обязательные параметры

`validateRecoveryParameters()` делает несколько нетривиальных вещей:

- в **non-standby archive recovery** требует GUC-переменную `restore_command`, иначе `FATAL`;
- в **standby archive recovery** допускает отсутствие GUC-переменных `primary_conninfo` и `restore_command`, но пишет `WARNING` и потом просто будет поллить `pg_wal`;
- если GUC-переменная `recovery_target_action = pause`, но GUC-переменная `hot_standby = off`, принудительно меняет action на `shutdown`;
- финально парсит GUC-переменную `recovery_target_time`, потому что ее интерпретация зависит от timezone;
- вычисляет или проверяет GUC-переменную `recovery_target_timeline`.

PITR - это startup-time поведение, важно различать две группы GUC.

- recovery target GUC-переменные, такие как `recovery_target`, `recovery_target_time`, `recovery_target_xid`, `recovery_target_name`, `recovery_target_lsn`, `recovery_target_timeline`, `recovery_target_action`, в коде объявлены как `PGC_POSTMASTER`, то есть требуют рестарта;
- отдельная standby/recovery GUC-переменная `hot_standby` тоже объявлена как `PGC_POSTMASTER`, но логически не относится к семейству `recovery_target*`;
- связанные standby/recovery GUC-переменные, такие как `restore_command` и `primary_conninfo`, в `guc_tables.c` объявлены как `PGC_SIGHUP`.

## Как выбирается стартовая точка replay

### Если есть `backup_label`

Если `read_backup_label()` находит `backup_label`, то PostgreSQL считает, что кластер поднимается из backup, и стартует не от `pg_control.checkPoint`, а от checkpoint, записанного в `backup_label`.

Это принципиально важно: `pg_control` мог уже уйти дальше, чем момент начала бэкапа, и если слепо стартовать от него, можно потерять консистентность backup chain.

В этом режиме `InitWalRecovery()`:

- сразу включает `InArchiveRecovery = true`;
- логирует `starting backup recovery with redo LSN ...`;
- читает checkpoint record по координатам из `backup_label`;
- проверяет, что `redo` действительно существует;
- применяет `tablespace_map`, если он присутствует;
- запоминает, что `backup_label` и `tablespace_map` потом надо переименовать в `*.old`.

### Если `backup_label` нет

Тогда сервер стартует от checkpoint из `pg_control`, но поведение зависит от контекста.

Если `ArchiveRecoveryRequested = true`, а сервер уже знает, где находится safe recovery boundary, например через:

- `minRecoveryPoint`
- `backupEndRequired`
- `backupEndPoint`
- или cluster state `DB_SHUTDOWNED`

то он может сразу войти в `archive recovery`.

Если же archive recovery requested, но до какого места надо докатиться до консистентности еще неизвестно, код сначала делает crash recovery по локально доступному WAL, и только потом продолжает archive recovery.

Это важный operational detail: не всякий PITR стартует "сразу из архива"; иногда код сначала проходит локальный хвост WAL.

## Как задается recovery target

Recovery targets реализованы через GUC hook-и в конце `xlogrecovery.c` и объявления GUC-переменных в `guc_tables.c`.

Ключевые факты:

- одновременно можно задать только один target;
- это enforced не описательно, а кодом: `assign_recovery_target*()` бросают ошибку `multiple recovery targets specified`;
- GUC-переменная `recovery_target` допускает только значение `immediate`;
- GUC-переменная `recovery_target_time` хранится строкой и парсится окончательно позже;
- GUC-переменная `recovery_target_timeline` допускает `current`, `latest` или numeric TLI;
- GUC-переменная `recovery_target_name` ограничена `MAXFNAMELEN - 1`;
- GUC-переменные `recovery_target_xid` и `recovery_target_lsn` парсятся и валидируются на этапе GUC assign/check.

Отдельно стоит отметить GUC-переменную `recovery_target_time`: в `check_recovery_target_time()` явно запрещены специальные значения `now`, `today`, `tomorrow`, `yesterday`. Код требует нормальный абсолютный timestamp.

### Что означает каждый target

- GUC-переменная `recovery_target = 'immediate'` - остановиться при достижении earliest consistent state;
- GUC-переменная `recovery_target_time` - остановиться по времени commit/abort записи;
- GUC-переменная `recovery_target_xid` - остановиться на transaction end record конкретного XID;
- GUC-переменная `recovery_target_lsn` - остановиться по LSN записи WAL;
- GUC-переменная `recovery_target_name` - остановиться на restore point с данным именем.

GUC-переменная `recovery_target_name` опирается на специальную WAL запись `XLOG_RESTORE_POINT`.

## Как создаются restore points

SQL-функция `pg_create_restore_point(text)` реализована в `src/backend/access/transam/xlogfuncs.c`.

Она:

- запрещена во время recovery;
- требует, чтобы GUC-переменная `wal_level` была как минимум `replica` или `logical`;
- ограничивает длину имени;
- вызывает `XLogRestorePoint()`.

`XLogRestorePoint()` в `src/backend/access/transam/xlog.c` создает WAL record типа `RM_XLOG_ID / XLOG_RESTORE_POINT`, записывая:

- `rp_time`
- `rp_name`

После этого recovery может найти эту запись и остановиться на ней через GUC-переменную `recovery_target_name`.

## Как PITR реально останавливает replay

Самая важная часть реализации, **core semantic PITR**, находится в `PerformWalRecovery()` в `xlogrecovery.c`.

Главный цикл устроен так:

1. читается очередная WAL запись;
2. вызывается `recoveryStopsBefore(xlogreader)`;
3. при необходимости делается `recoveryApplyDelay()`;
4. запись применяется через `ApplyWalRecord(...)`;
5. вызывается `recoveryStopsAfter(xlogreader)`;
6. затем читается следующая запись.

PostgreSQL имеет две разные точки принятия решения:

- остановка до применения записи;
- остановка после применения записи.

## Семантика остановки по типам target

### GUC: `recovery_target_time`

`recoveryStopsBefore()` получает timestamp из commit/abort record через `getRecordTimestamp()`.

Дальше логика такая:

- если GUC-переменная `recovery_target_inclusive = on`, startup останавливается перед первой transaction end record, у которой `recordXtime > recoveryTargetTime`;
- если GUC-переменная `recovery_target_inclusive = off`, startup останавливается перед первой transaction end record, у которой `recordXtime >= recoveryTargetTime`.

Следствие:

- inclusive mode включает все транзакции, завершившиеся ровно в target time;
- exclusive mode отрезает их.

### GUC: `recovery_target_xid`

Для `XID` код работает не по диапазону, а по строгому равенству:

- exclusive stop реализован в `recoveryStopsBefore()`;
- inclusive stop реализован в `recoveryStopsAfter()`.

В коде есть комментарий, почему нельзя сравнивать `<=` или `>=`: `xid` выдаются в порядке старта транзакций, а завершаются не в этом порядке. Поэтому PITR по `xid` корректно определяется только как "найти transaction end record именно этого XID".

### GUC: `recovery_target_lsn`

Для `LSN` логика простая:

- exclusive - остановка в `recoveryStopsBefore()` при `record->ReadRecPtr >= recoveryTargetLSN`;
- inclusive - остановка в `recoveryStopsAfter()` при том же условии.

LSN target определяет, включать ли найденную запись в replay.

### GUC: `recovery_target_name`

GUC-переменная `recovery_target_name` обрабатывается только в `recoveryStopsAfter()`:

- проверяет, что запись относится к `RM_XLOG_ID`;
- проверяет `info == XLOG_RESTORE_POINT`;
- сравнивает `rp_name` с `recoveryTargetName`;
- останавливается после применения первой совпавшей restore point записи.

Если restore point с таким именем встречается несколько раз, код останавливается на первой найденной.

### GUC: `recovery_target = 'immediate'`

Этот режим привязан к флагу `reachedConsistency`.

`reachedConsistency` становится `true`, когда:

- recovery уже не ждет `backup end`;
- `minRecoveryPoint <= lastReplayedEndRecPtr`;
- успешно пройдены проверки `XLogCheckInvalidPages()` и `CheckTablespaceDirectory()`.

После этого код логирует `consistent recovery state reached at ...`.

Для GUC-переменной `recovery_target = 'immediate'` сервер останавливается, как только consistent state достигнут:

- если он был достигнут ранее, stop происходит в `recoveryStopsBefore()`;
- если именно текущая запись довела сервер до consistency, stop фиксируется в `recoveryStopsAfter()`.

Смысл этого режима: как можно раньше дойти до консистентного состояния и остановиться.

## Какие WAL-записи вообще участвуют в decision-making

В PITR stop logic сервер не анализирует каждую WAL запись одинаково.

Основные decision points:

- `RM_XACT_ID` commit/abort records для `time` и `xid`;
- `RM_XLOG_ID / XLOG_RESTORE_POINT` для `name`;
- `record->ReadRecPtr` для `lsn`.

Recovery target означает остановку в специально определенных точках WAL stream.

## Что происходит, если target недостижим

Если main redo loop закончился, а `reachedRecoveryTarget` так и не стал `true`, то в конце `PerformWalRecovery()` сервер завершает startup ошибкой:

- `recovery ended before configured recovery target was reached`

Если target найден, но он лежит раньше `consistent recovery point`, startup завершается с:

- `requested recovery stop point is before consistent recovery point`

PostgreSQL не позволит остановиться в точке, где кластер еще не консистентен.

## Как PITR взаимодействует с timeline

Во время startup:

- `validateRecoveryParameters()` вычисляет `recoveryTargetTLI`;
- при значении GUC-переменной `recovery_target_timeline = 'latest'` код ищет newest timeline через history files;
- затем `InitWalRecovery()` проверяет, что checkpoint и `minRecoveryPoint` действительно принадлежат истории запрошенной timeline.

При завершении archive recovery / promotion path в `StartupXLOG()` PostgreSQL создает новую timeline:

- `newTLI = findNewestTimeLine(recoveryTargetTLI) + 1`;
- пишет history file через `writeTimeLineHistory(...)`;
- логирует `selected new timeline ID: ...`;
- удаляет `recovery.signal` и `standby.signal`, чтобы при следующем старте не войти в archive recovery повторно по ошибке.

Практический смысл:

- PITR обычно приводит к fork истории WAL при завершении recovery / promotion;
- после recovery получаем новую ветку timeline;
- старую timeline можно потом использовать как альтернативную ветку истории.

PITR обычно приводит к **новой точке ветвления истории** (а не просто докатывает архив и запускается).

## Что происходит в конце recovery

После завершения replay `StartupXLOG()` делает еще несколько критичных шагов:

- завершает `FinishWalRecovery()`;
- при archive recovery и promotion может записать `end-of-recovery record` через `CreateEndOfRecoveryRecord()`;
- в противном случае инициирует `CHECKPOINT_END_OF_RECOVERY`;
- очищает recovery signal files;
- переводит сервер из `InRecovery = true` в production mode;
- выполняет `CleanupAfterArchiveRecovery(...)`.

`PerformRecoveryXLogAction()` в `xlog.c` показывает важную развилку:

- если recovery заканчивается promotion под postmaster, вместо полного checkpoint может быть записан специальный end-of-recovery record;
- иначе делается полноценный checkpoint конца recovery.

## GUC: `recovery_target_action`

Когда recovery дошел до target, `PerformWalRecovery()` смотрит на значение GUC-переменной `recovery_target_action`:

- `shutdown` - процесс startup выходит со специальным кодом, чтобы postmaster завершил сервер;
- `pause` - recovery ставится на паузу через `SetRecoveryPause(true)` и ждет `pg_wal_replay_resume()`;
- `promote` - recovery завершается и сервер продолжает переход в normal production.

Два разных слоя логики: "дойти до target" и "что делать после target".

## Минимальная конфигурационная модель

Для обычного PITR без standby mode минимально нужны:

```conf
restore_command = 'cp /path/to/archive/%f %p'
recovery_target_time = '2026-04-21 10:35:00+03'
recovery_target_timeline = 'latest'
recovery_target_action = 'pause'
```

И файл:

```text
$PGDATA/recovery.signal
```

Важно:

- без `recovery.signal` или `standby.signal` archive recovery не стартует;
- без GUC-переменной `restore_command` non-standby PITR не пройдет валидацию;
- задавать можно только одну GUC-переменную из семейства `recovery_target*`;
- GUC-переменная `recovery_target_inclusive` меняет семантику "включить target record или нет";
- GUC-переменная `recovery_target = 'immediate'` не эквивалентна GUC-переменной `recovery_target_time = 'now'`, и код прямо запрещает такие relative timestamps.

## Выводы

- PITR в PostgreSQL реализован как набор правил поверх общего WAL recovery pipeline.
- Точка остановки определяется конкретными WAL record boundary.
- `time` и `xid` targets в основном опираются на transaction end records.
- `name` target опирается на специальные `XLOG_RESTORE_POINT` записи.
- `lsn` target работает на уровне физических координат WAL stream.
- Консистентность проверяется отдельно от достижения target, и target "слишком рано" недопустим.
- При завершении archive recovery / promotion PostgreSQL создает новую timeline, то есть PITR обычно приводит к fork истории WAL.

## Выводы кратко

PITR в коде PostgreSQL - это:

1. войти в `archive recovery`;
2. выбрать стартовый checkpoint из `backup_label` или `pg_control`;
3. читать WAL по выбранной timeline с учетом timeline history;
4. после каждой записи проверять, не достигнут ли recovery target;
5. при достижении target корректно завершить recovery;
6. при завершении recovery / promotion создать новую timeline и перевести сервер в новое состояние.
