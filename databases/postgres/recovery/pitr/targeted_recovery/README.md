# PostgreSQL Targeted Recovery: How It Works

The relevant code paths are primarily:

- `src/include/access/xlogrecovery.h`
- `src/backend/access/transam/xlogrecovery.c`

## What targeted recovery is for

Targeted recovery is a **Point-In-Time Recovery (PITR)** mechanism that allows PostgreSQL to stop replaying WAL not just at the latest available point, but at a specific logical stop condition chosen by the user.

The intent is straightforward:

- replay WAL until a chosen stop point is reached
- stop recovery at that point
- expose a database state that is **consistent** with that stop point

Conceptually, this means:

- all changes that are supposed to be visible before the stop point must already be applied and visible
- all changes after the stop point must not yet be visible

Otherwise the recovery target loses its meaning: PostgreSQL may report that it stopped at the requested point, while the actual visible database state does not correspond to that point.

This functionality is used for cases such as:

- recovering to a specific transaction boundary
- recovering to a specific timestamp
- recovering to a named restore point
- recovering to a precise WAL location
- recovering only until the server reaches a consistent state

## Recovery target types in PostgreSQL

The recovery target types are defined in `src/include/access/xlogrecovery.h`:

```c
typedef enum
{
	RECOVERY_TARGET_UNSET,
	RECOVERY_TARGET_XID,
	RECOVERY_TARGET_TIME,
	RECOVERY_TARGET_NAME,
	RECOVERY_TARGET_LSN,
	RECOVERY_TARGET_IMMEDIATE,
} RecoveryTargetType;
```

Important note from the same header:

- recovery targets are only used during Point-In-Time recovery
- they are not used for normal standby mode replay

PostgreSQL supports these target modes:

### `RECOVERY_TARGET_XID`

Stop recovery at a specific transaction ID.

The relevant logic is in `recoveryStopsBefore()` and `recoveryStopsAfter()` in `xlogrecovery.c`.

Semantics depend on `recovery_target_inclusive`:

- if inclusive is `off`, PostgreSQL stops before the matching transaction end record
- if inclusive is `on`, PostgreSQL stops after the matching transaction end record

For `XID`, PostgreSQL intentionally compares for equality only, because transaction IDs are assigned in start order, not completion order.

### `RECOVERY_TARGET_TIME`

Stop recovery at a specific timestamp.

This is also checked in both `recoveryStopsBefore()` and `recoveryStopsAfter()`, using the timestamp extracted from commit/abort records.

Semantics depend on `recovery_target_inclusive`:

- inclusive `on`: stop after the last transaction whose timestamp is still considered part of the target state
- inclusive `off`: stop before the first transaction at or beyond the target

### `RECOVERY_TARGET_NAME`

Stop recovery at a named restore point.

This is handled in `recoveryStopsAfter()` by looking for an `RM_XLOG_ID / XLOG_RESTORE_POINT` record and comparing the restore point name.

PostgreSQL stops at the first restore point with the requested name.

### `RECOVERY_TARGET_LSN`

Stop recovery at a specific WAL location.

This is checked against `record->ReadRecPtr`.

Semantics again depend on `recovery_target_inclusive`:

- inclusive `off`: stop before the target LSN in `recoveryStopsBefore()`
- inclusive `on`: stop after reaching the target LSN in `recoveryStopsAfter()`

### `RECOVERY_TARGET_IMMEDIATE`

Stop recovery as soon as the database reaches a consistent recovery state.

This is a special case used when the goal is not a business-level point in time, but simply "stop as soon as it is safe to open the database".

The checks for this exist in both `recoveryStopsBefore()` and `recoveryStopsAfter()` and depend on `reachedConsistency`.

## How PostgreSQL decides when to stop

The main decision points are two functions in `src/backend/access/transam/xlogrecovery.c`:

- `recoveryStopsBefore(XLogReaderState *record)`
- `recoveryStopsAfter(XLogReaderState *record)`

These two functions exist because some targets are defined naturally:

- before applying the current record
- after applying the current record

This distinction is important for correctness.

Examples:

- `RECOVERY_TARGET_LSN` with inclusive `off` is checked before applying the
  record
- `RECOVERY_TARGET_NAME` is checked after applying the restore-point record
- `RECOVERY_TARGET_XID` may stop before or after the matching transaction end
  depending on `recovery_target_inclusive`

When a stop condition is met, PostgreSQL stores metadata describing why replay stopped:

- transaction ID
- timestamp
- LSN
- restore point name
- whether it stopped before or after the boundary

This information is later used to describe the recovery stop reason, for example in timeline history handling.

## Inclusive vs exclusive semantics

PostgreSQL exposes `recoveryTargetInclusive` as a user-configurable behavior.

This setting answers a simple question:

- should the database state include the target record or stop just before it?

This is implemented differently for different target types:

- for `XID`, PostgreSQL distinguishes before/after the transaction end record
- for `TIME`, PostgreSQL compares the commit/abort timestamp with the target timestamp
- for `LSN`, PostgreSQL decides whether to stop before or after the WAL location

The result is tracked through `recoveryStopAfter`, which becomes part of the final stop reason and end-of-recovery metadata.

## Custom WAL and extension hook support

PostgreSQL includes an extension hook for custom WAL resource managers:

```c
typedef bool (*RecoveryStopsBeforeHookType) (XLogReaderState *record,
											 TransactionId *recordXid,
											 TimestampTz *recordXtime);
```

This hook is checked inside `recoveryStopsBefore()` when PostgreSQL encounters a custom rmgr record:

- if the record belongs to a custom resource manager
- and `RecoveryStopsBeforeHook` is installed
- PostgreSQL delegates the "should we stop before this record?" decision to
  the extension

This is important because PostgreSQL itself only understands built-in WAL formats. If an extension stores its own transaction/timestamp semantics inside custom WAL records, PostgreSQL cannot infer recovery target behavior for those records unless the extension explicitly teaches it how.

This is especially relevant for time-based recovery over custom WAL.

## What happens after PostgreSQL decides to stop

The flow after a recovery target is reached.

In the main replay loop PostgreSQL first:

- detects that the recovery target has been reached
- applies `recovery_target_action` (`shutdown`, `pause`, or `promote`)
- and then, depending on that action, may proceed further toward full end-of-recovery completion

This distinction matters because `target reached` and `recovery fully completed` are not the same event.

The broader end-of-recovery flow later goes through `FinishWalRecovery()` in `xlogrecovery.c`.

This function is responsible for end-of-recovery tasks such as:

- shutting down WAL receiver state
- leaving archive recovery mode
- determining the exact end-of-log position
- preparing the system for post-recovery WAL generation

Separately, the main replay loop also executes resource-manager cleanup after the target condition is reached.

This matters because there is a real distinction between:

- PostgreSQL has decided that replay should stop here
- all extension-specific asynchronous replay machinery is fully finished

For built-in PostgreSQL state, these are naturally aligned by the recovery implementation. For custom extensions with their own parallel or asynchronous redo model, this alignment has to be ensured explicitly by the extension.

## Why this matters for Oriole

The PostgreSQL side assumes that when recovery stops at a target point, the visible database state is already consistent with that point.

That assumption is valid for PostgreSQL's own recovery machinery.

However, if an extension:

- replays part of its state asynchronously
- finalizes visibility later than PostgreSQL's replay stop decision

then PostgreSQL can report that the target has been reached while the extension-visible state is still lagging behind.
