# Notification Service

## Overview

The notification service is the platform's asynchronous email worker. It consumes Celery tasks from RabbitMQ and sends transactional emails for signup and order lifecycle events.

## Purpose

- Consume signup and order-related events from RabbitMQ
- Send email through SMTP
- Retry failed email tasks automatically
- Expose a minimal health endpoint

## Location

- Source: `sales-notification-service/`
- FastAPI entrypoint: `sales-notification-service/app/main.py`
- Celery app: `sales-notification-service/app/core/celery_app.py`
- Worker bootstrap: `sales-notification-service/entrypoint.sh`
- Email tasks: `sales-notification-service/app/tasks/email_tasks.py`, `sales-notification-service/app/tasks/order_email_tasks.py`
- SMTP service helper: `sales-notification-service/app/services/email_service.py`
- Docker build: `sales-notification-service/Dockerfile`
- Local compose service: `docker-compose.yaml`
- Kubernetes deployment: `k8s/Deployments/notification-deployment.yaml`

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
| GET | `/api/notification/health` | Health check | No | None |

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

Secret name:

- `notification-secret`

Required secret keys:

- `RABBITMQ_URL`
- `FROM_EMAIL`
- `FROM_NAME`
- `SMTP_HOST`
- `SMTP_PASS`
- `SMTP_PORT`
- `SMTP_USER`

Required ConfigMap keys:

- None

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

## Roles and permissions

- No RBAC or JWT enforcement is implemented here.
- This service trusts upstream producers and environment configuration rather than end-user tokens.

## Current limitations

- No persistence layer exists for notification history.
- Invoice or payment notification tasks are not implemented in the current workspace.

## GitOps workflow

If you want to learn more about the GitOps workflow documentation of this microservice, go to this link: `<gitops-doc-link-for-notification-service>`
