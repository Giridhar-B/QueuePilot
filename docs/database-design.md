# QueuePilot – Database Design

## 1. Overview

QueuePilot uses PostgreSQL as its primary relational database and source of truth.

The database stores client accounts, API credentials, jobs, execution attempts, schedules, and transactional outbox events.

The schema is designed to support:

- Durable job storage
- Multi-account ownership and isolation
- Job execution history
- Retry management
- Idempotent job submission
- Persistent scheduling
- Reliable queue publication
- Worker crash recovery

## 2. Database Tables

### 2.1 accounts

Stores client accounts using QueuePilot.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | Primary Key | Unique account identifier |
| name | VARCHAR(150) | NOT NULL | Account name |
| created_at | TIMESTAMPTZ | NOT NULL | Account creation time |
| updated_at | TIMESTAMPTZ | NOT NULL | Last update time |

### 2.2 api_keys

Stores API credentials associated with client accounts.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | Primary Key | API key identifier |
| account_id | UUID | Foreign Key | Associated account |
| name | VARCHAR(100) | NOT NULL | API key name |
| key_prefix | VARCHAR(20) | NOT NULL | Non-secret identification prefix |
| key_hash | VARCHAR(255) | NOT NULL, UNIQUE | Securely hashed API key |
| expires_at | TIMESTAMPTZ | NULL | Optional expiration |
| revoked_at | TIMESTAMPTZ | NULL | Revocation time |
| created_at | TIMESTAMPTZ | NOT NULL | Creation time |

Raw API keys will only be shown when generated. They will never be stored in the database.

### 2.3 jobs

Stores submitted background jobs and their current execution state.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | Primary Key | Unique job identifier |
| account_id | UUID | Foreign Key | Job owner |
| schedule_id | UUID | Nullable Foreign Key | Source schedule, if applicable |
| job_type | VARCHAR(100) | NOT NULL | Handler type |
| status | VARCHAR(30) | NOT NULL | Current job status |
| priority | INTEGER | NOT NULL | Execution priority |
| payload | JSONB | NOT NULL | Job input data |
| idempotency_key | VARCHAR(255) | NULL | Client submission key |
| attempt_count | INTEGER | NOT NULL | Number of started attempts |
| max_attempts | INTEGER | NOT NULL | Maximum allowed attempts |
| available_at | TIMESTAMPTZ | NOT NULL | Earliest eligible execution time |
| timeout_seconds | INTEGER | NOT NULL | Execution timeout |
| worker_id | VARCHAR(150) | NULL | Current worker identifier |
| lease_expires_at | TIMESTAMPTZ | NULL | Execution lease expiration |
| heartbeat_at | TIMESTAMPTZ | NULL | Last worker heartbeat |
| result | JSONB | NULL | Successful execution result |
| last_error | TEXT | NULL | Latest failure information |
| created_at | TIMESTAMPTZ | NOT NULL | Submission time |
| updated_at | TIMESTAMPTZ | NOT NULL | Last update time |
| completed_at | TIMESTAMPTZ | NULL | Completion time |

Idempotency keys will be unique within an account when provided.

### 2.4 job_attempts

Stores the execution history of each job.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | Primary Key | Attempt identifier |
| job_id | UUID | Foreign Key | Associated job |
| attempt_number | INTEGER | NOT NULL | Attempt sequence |
| worker_id | VARCHAR(150) | NOT NULL | Executing worker |
| status | VARCHAR(30) | NOT NULL | Attempt outcome |
| started_at | TIMESTAMPTZ | NOT NULL | Execution start |
| finished_at | TIMESTAMPTZ | NULL | Execution end |
| duration_ms | BIGINT | NULL | Execution duration |
| error_message | TEXT | NULL | Failure details |
| error_type | VARCHAR(150) | NULL | Failure category |

Each execution attempt is recorded separately, allowing users to inspect job history.

### 2.5 schedules

Stores persistent one-time and recurring job schedules.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | Primary Key | Schedule identifier |
| account_id | UUID | Foreign Key | Schedule owner |
| name | VARCHAR(150) | NOT NULL | Schedule name |
| job_type | VARCHAR(100) | NOT NULL | Handler type |
| payload | JSONB | NOT NULL | Job input |
| schedule_type | VARCHAR(30) | NOT NULL | ONE_TIME or RECURRING |
| cron_expression | VARCHAR(100) | NULL | Recurrence expression |
| timezone | VARCHAR(100) | NOT NULL | Schedule timezone |
| next_run_at | TIMESTAMPTZ | NULL | Next execution time |
| last_run_at | TIMESTAMPTZ | NULL | Previous execution time |
| start_at | TIMESTAMPTZ | NULL | Optional start boundary |
| end_at | TIMESTAMPTZ | NULL | Optional end boundary |
| status | VARCHAR(30) | NOT NULL | Schedule state |
| created_at | TIMESTAMPTZ | NOT NULL | Creation time |
| updated_at | TIMESTAMPTZ | NOT NULL | Last update time |

The scheduler creates ordinary job records when schedules become due. Scheduled jobs use the same worker execution pipeline as manually submitted jobs.

### 2.6 outbox_events

Stores events that must be published to Redis Streams.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | Primary Key | Event identifier |
| job_id | UUID | Foreign Key | Associated job |
| event_type | VARCHAR(100) | NOT NULL | Event category |
| payload | JSONB | NOT NULL | Event data |
| created_at | TIMESTAMPTZ | NOT NULL | Event creation time |
| published_at | TIMESTAMPTZ | NULL | Successful publication time |
| lease_owner | VARCHAR(150) | NULL | Current dispatcher |
| lease_expires_at | TIMESTAMPTZ | NULL | Dispatcher lease expiration |
| publish_attempts | INTEGER | NOT NULL | Publication attempts |
| last_error | TEXT | NULL | Latest publication failure |

Outbox events are inserted in the same PostgreSQL transaction as the corresponding job record.

The dispatcher publishes eligible events to Redis Streams and marks them as published after successful publication.

## 3. Entity Relationships

- One account can own multiple API keys.
- One account can submit multiple jobs.
- One account can own multiple schedules.
- One job can have multiple execution attempts.
- One schedule can generate multiple jobs.
- One job can produce multiple outbox events.

## 4. Important Database Constraints

The database will enforce:

- Primary keys on all tables.
- Foreign key relationships between related records.
- Unique API key hashes.
- Unique idempotency keys per account when provided.
- Unique attempt numbers per job.
- Non-negative attempt counts.
- Valid job status values.
- Valid schedule types and statuses.
- Positive timeout values and maximum attempt counts.

## 5. Important Indexes

We will create indexes for common query patterns:

- Jobs by account and creation time.
- Jobs by account and status.
- Jobs eligible for execution, using status and available_at.
- Jobs with expired execution leases.
- Job attempts by job ID and attempt number.
- Schedules by status and next_run_at.
- Unpublished outbox events.
- API keys by key hash.

## 6. Database Design Principles

1. PostgreSQL is the authoritative source of job state.
2. Redis is not used as the permanent job record store.
3. Job attempts are stored separately from the main job record.
4. Account ownership is enforced in database queries and application authorization.
5. JSONB is used for flexible task-specific payloads and results.
6. Database transactions protect critical state changes.
7. Indexes support efficient job retrieval and background processing.
8. Timestamps use TIMESTAMPTZ to avoid timezone ambiguity.

## 7. Future Database Considerations

The initial schema will be refined during implementation.

Potential future additions include:

- Job tags and metadata.
- Job dependencies.
- Dead-letter handling.
- Per-account quotas.
- Audit logs.
- Worker registration and health information.