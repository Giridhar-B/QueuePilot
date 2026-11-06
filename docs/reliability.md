# QueuePilot – Reliability Architecture

## 1. Overview

QueuePilot is designed to process background jobs reliably, even when individual components fail.

The reliability architecture focuses on:

- Preventing job loss
- Handling duplicate message delivery
- Recovering from worker crashes
- Retrying failed executions
- Managing execution timeouts
- Coordinating concurrent workers
- Maintaining consistent job state

The system uses PostgreSQL as its source of truth and Redis Streams as its message transport layer.

## 2. Job Lifecycle

A job moves through the following states:

| State | Description |
|---|---|
| QUEUED | Job is waiting for execution |
| RUNNING | A worker is executing the job |
| RETRY_WAIT | Failed job is waiting for its next retry |
| SUCCEEDED | Job completed successfully |
| FAILED | Job exhausted its retry attempts or encountered a terminal failure |
| CANCELLED | Job was cancelled before execution |

### State Transitions

- QUEUED → RUNNING
- RUNNING → SUCCEEDED
- RUNNING → RETRY_WAIT
- RETRY_WAIT → QUEUED
- RUNNING → FAILED
- QUEUED → CANCELLED
- RETRY_WAIT → CANCELLED

A job in RUNNING state cannot be assumed to have stopped merely because a cancellation request was received. Execution cancellation requires a separate cooperative cancellation mechanism.

## 3. Retry Mechanism

When a job fails, QueuePilot determines whether another execution attempt is permitted.

Retry configuration includes:

- Maximum number of attempts
- Initial retry delay
- Maximum retry delay
- Exponential backoff multiplier

The retry delay follows an exponential backoff strategy:

delay = min(initialDelay × 2^(attemptNumber - 1), maximumDelay)

Jitter will be added to retry delays to prevent many failed jobs from retrying simultaneously.

If the maximum number of attempts is exhausted, the job transitions to FAILED.

## 4. Worker Failure Recovery

A worker may crash while processing a job because of:

- Application failure
- Machine shutdown
- Container termination
- Network interruption
- Database connectivity issues

QueuePilot uses execution leases to identify abandoned jobs.

Each worker claim records:

- Job ID
- Worker ID
- Lease expiration time
- Execution attempt information

Workers periodically renew their leases while executing jobs.

A recovery process identifies expired leases and determines whether the job should be retried or marked as failed.

## 5. Duplicate Message Handling

Redis Streams provides at-least-once message delivery.

A message may be delivered again when:

- A worker crashes before acknowledging it
- A pending message is reclaimed
- Queue reconciliation republishes a job event

QueuePilot uses database-backed job claiming to prevent multiple workers from executing the same active job concurrently.

Job state transitions and claims must be performed atomically.

Duplicate delivery must not create duplicate execution attempts for the same active claim.

## 6. Transactional Outbox Pattern

QueuePilot uses the transactional outbox pattern to coordinate PostgreSQL and Redis.

When a job is submitted:

1. The job record is inserted into PostgreSQL.
2. An outbox event is inserted into PostgreSQL.
3. Both operations occur in the same database transaction.
4. The outbox dispatcher reads unpublished events.
5. The dispatcher publishes events to Redis Streams.
6. Successfully published events are marked as published.

If Redis is unavailable, the outbox event remains available for later publication.

If the dispatcher crashes after publishing but before marking an event as published, the event may be published again. Workers must safely handle duplicate delivery.

## 7. Idempotency

QueuePilot supports idempotency keys for job submission.

An idempotency key allows a client to retry a submission request without unintentionally creating multiple job records.

The system also uses job identifiers and execution claims to prevent duplicate concurrent processing.

However, external side effects such as sending emails or delivering webhooks may still happen more than once if a worker crashes after the external operation succeeds but before recording success.

Exactly-once external execution is not guaranteed.

## 8. Timeout Management

Jobs can define an execution timeout.

If execution exceeds the configured timeout:

- The execution is marked as timed out.
- The attempt is recorded as failed.
- The retry policy is evaluated.
- The job is retried or marked as FAILED.

Java thread interruption alone cannot guarantee that an external operation has stopped. Handlers must use bounded network timeouts and cooperative cancellation where possible.

## 9. Queue Reconciliation

QueuePilot periodically compares PostgreSQL job state with Redis queue publication state.

The reconciliation process identifies:

- Queued jobs without a published queue event
- Outbox events that remain unpublished
- Jobs whose execution leases have expired
- Inconsistent processing states

Eligible jobs are republished or recovered without creating duplicate active executions.

PostgreSQL remains the source of truth throughout reconciliation.

## 10. Reliability Guarantees

QueuePilot aims to provide:

- Durable job persistence
- At-least-once message delivery
- Database-backed execution coordination
- Configurable retry policies
- Worker crash recovery
- Duplicate delivery protection
- Idempotent job submission
- Persistent scheduling
- Execution history and failure visibility

The system does not guarantee exactly-once execution of external side effects.