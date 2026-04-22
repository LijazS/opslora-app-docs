# Notification Service

## Overview

The notification service is the platform's asynchronous email worker. It consumes Celery tasks from RabbitMQ and sends transactional emails for signup and order lifecycle events.

## Purpose

- Consume signup and order-related events from RabbitMQ
- Send email through SMTP
- Retry failed email tasks automatically
- Expose a minimal health endpoint

## Features

- RabbitMQ-backed task consumption
- SMTP email delivery with STARTTLS
- Automatic retries with exponential backoff
- Separate task handlers for signup and order events
- Global Celery task-failure logging

## API surface

### HTTP endpoint

| Method | Path | Purpose | Auth | Permission |
| --- | --- | --- | --- | --- |
| GET | `/api/v1/notification/health` | Health check | No | None |

### Celery task surface

| Task name | Triggered by | Purpose |
| --- | --- | --- |
| `notification.send_signup_email` | Auth service | Welcome email after signup |
| `notification.send_order_created_email` | Order service | Customer email when order is created |
| `notification.send_order_confirmed_email` | Order service | Customer email when order is confirmed |
| `notification.send_order_cancelled_email` | Order service | Customer email when order is cancelled |

## Communication with other services

### Inbound

- Auth service publishes signup notifications to `notification_queue`
- Order service publishes order lifecycle notifications to `notification_queue`
- RabbitMQ provides the transport

### Outbound

- SMTP server defined by:
  - `SMTP_HOST`
  - `SMTP_PORT`
  - `SMTP_USER`
  - `SMTP_PASS`
  - `FROM_EMAIL`
  - `FROM_NAME`

## Deployment config and secrets

- Secret name: `notification-secret`
- Required secret keys:
  - `RABBITMQ_URL`
  - `FROM_EMAIL`
  - `FROM_NAME`
  - `SMTP_HOST`
  - `SMTP_PASS`
  - `SMTP_PORT`
  - `SMTP_USER`
- Required ConfigMap keys: None

## Dockerfile

The Notification Service uses a multi-stage Python Docker build.

Security approach:

- The runtime image uses `dhi.io/python:3.13.13`, which is a hardened base image intended for safer production use.
- Hardened images help reduce unnecessary packages and lower the attack surface of the container.
- Using a curated runtime base also supports stronger security and compliance practices compared to a generic development-oriented image.
- The multi-stage build keeps dependency installation in the builder stage and leaves the runtime image smaller and cleaner.
- The container runs as non-root user `10001`, which limits the impact of a container breakout or application-level compromise.
- File copies use controlled ownership and permissions so the runtime container has only the access it needs.

Build stages:

- Builder stage:
  - Starts from `dhi.io/python:3.13-dev`
  - Creates a virtual environment at `/app/venv`
  - Copies `requirements.txt`
  - Installs Python dependencies into the virtual environment
  - Uses a pip cache mount to speed up repeated builds
- Runtime stage:
  - Starts from `dhi.io/python:3.13.13`
  - Copies the prepared virtual environment from the builder stage
  - Copies the `app/` source directory into the image

Runtime configuration:

- Working directory: `/app`
- `PATH` includes `/app/venv/bin`
- `PYTHONUNBUFFERED=1` is enabled
- Runs as non-root user `10001`
- Exposes port `3000`
- Starts the service with `uvicorn app.main:app --host 0.0.0.0 --port 3000`

## Message payloads

### Signup email payload

- `payload.user_id`
- `payload.email`
- `payload.organization_name`

### Order email payload

- `payload.order_id`
- `payload.customer_id`
- `payload.organization_id`
- `payload.email`
- `payload.customer_name`
- `payload.items[]`

## Operational behavior

- Worker waits for RabbitMQ before starting.
- Worker consumes from `notification_queue`.
- Concurrency is set to `2` in `entrypoint.sh`.
- Each task retries up to `5` times on exception.
- FastAPI is present only for health checking; the main workload is the Celery worker process.
- The HTTP health endpoint uses the `/api/v1` prefix.

## Roles and permissions

- No RBAC or JWT enforcement is implemented here.
- This service trusts upstream producers and environment configuration rather than end-user tokens.

## Current limitations

- No persistence layer exists for notification history.
- Invoice or payment notification tasks are not implemented in the current workspace.

## GitOps workflow

If you want to learn more about the GitOps workflow documentation of this microservice, go to this link: `<gitops-doc-link-for-notification-service>`
