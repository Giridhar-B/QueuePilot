# QueuePilot – System Architecture

## 1. Project Overview

QueuePilot is a distributed background job processing and orchestration platform built using Java, Spring Boot, PostgreSQL, and Redis Streams.

It allows client applications to submit background jobs through REST APIs. These jobs are stored durably, queued for processing, and executed asynchronously by background workers.

The platform provides:

- Asynchronous job processing
- Reliable job delivery
- Configurable retry mechanisms
- Exponential backoff
- Job prioritization
- Worker concurrency management
- Job scheduling
- Failure recovery
- Job execution history
- API key authentication
- Monitoring through a web dashboard

## 2. Technology Stack

| Component | Technology |
|---|---|
| Backend | Java 21, Spring Boot 3.x |
| Build Tool | Maven |
| Database | PostgreSQL |
| Database Migration | Flyway |
| Message Queue | Redis Streams |
| ORM | Spring Data JPA / Hibernate |
| Security | Spring Security |
| Frontend | React, TypeScript, Vite |
| Styling | Tailwind CSS |
| API Documentation | OpenAPI / Swagger |
| Testing | JUnit 5, Mockito, Testcontainers |
| Containerization | Docker, Docker Compose |

## 3. System Components

### 3.1 REST API

Responsible for:

- Accepting job submissions
- Authenticating API clients
- Validating incoming requests
- Managing job status
- Supporting job cancellation and manual retries
- Managing scheduled jobs

### 3.2 PostgreSQL

Acts as the primary source of truth.

Stores:

- Job records
- Job execution attempts
- Scheduled jobs
- API keys (hashed)
- Outbox events
- Execution metadata

### 3.3 Redis Streams

Responsible for:

- Delivering job messages to workers
- Managing consumer groups
- Tracking pending messages
- Supporting message acknowledgement

Redis is used as the message transport layer. PostgreSQL remains the durable source of truth.

### 3.4 Outbox Dispatcher

Responsible for:

- Reading unpublished outbox events from PostgreSQL
- Publishing job messages to Redis Streams
- Retrying failed message publication
- Marking successfully published events

### 3.5 Background Workers

Responsible for:

- Consuming messages from Redis Streams
- Claiming jobs for execution
- Executing the appropriate task handler
- Recording execution attempts
- Managing retries and timeouts
- Updating job status
- Acknowledging processed messages

### 3.6 Task Handlers

QueuePilot initially supports:

- Email processing
- PDF generation
- Webhook delivery
- Image processing

Each handler implements a common execution contract so new task types can be added without changing the core worker pipeline.

### 3.7 Scheduler

Responsible for:

- Managing one-time scheduled jobs
- Managing recurring jobs
- Identifying due schedules
- Creating normal jobs for execution through the existing processing pipeline

### 3.8 React Dashboard

Provides an interface to:

- Submit jobs
- View job status
- Inspect execution history
- Monitor failed jobs
- Retry eligible jobs
- Cancel eligible jobs
- Manage scheduled jobs
- Monitor system health

## 4. High-Level Job Processing Flow

1. A client submits a job through the REST API.
2. The API validates and authenticates the request.
3. The job and its outbox event are saved in PostgreSQL within a database transaction.
4. The outbox dispatcher publishes the event to Redis Streams.
5. A background worker consumes the message.
6. The worker claims the job and executes its handler.
7. The execution result and attempt history are persisted in PostgreSQL.
8. The worker acknowledges the Redis message.

## 5. Reliability Principles

QueuePilot follows these principles:

- PostgreSQL is the source of truth.
- Redis Streams provides message delivery.
- The transactional outbox pattern prevents jobs from being lost between database persistence and queue publication.
- Workers use safe job-claiming mechanisms to coordinate concurrent execution.
- Failed jobs can be retried with exponential backoff.
- Abandoned executions can be recovered.
- Duplicate message delivery is expected and handled through idempotency protections.
- The system provides at-least-once processing, not exactly-once external side effects.

## 6. Deployment Architecture

The initial deployment uses Docker Compose to run:

- Spring Boot backend
- PostgreSQL
- Redis
- React frontend

The backend initially runs as a modular monolith. API handling, scheduling, dispatching, and worker execution are separate logical components within the application.

This architecture allows us to maintain clear module boundaries while keeping initial deployment and development manageable.