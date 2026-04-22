# Auth Service

## Overview

The auth service is the identity and authorization entry point for the platform. It creates organizations, creates the first user for an organization, assigns the `OWNER` role, seeds the RBAC catalog, and issues JWT access tokens used by the rest of the platform.

## Purpose

- Register a new organization workspace.
- Register the first user in that organization.
- Authenticate existing users.
- Seed roles and permission definitions.
- Publish signup notification events to RabbitMQ.

## Responsibilities

- Create `organizations`, `users`, `organization_users`, and `user_roles` records during signup.
- Hash passwords with bcrypt-compatible helpers in `app/security/password.py`.
- Generate JWTs with `user_id`, `email`, `org_id`, `permissions`, and `exp`.
- Resolve permissions by joining organization membership, roles, and role-permission mappings.
- Publish `notification.send_signup_email` tasks to the `notification_queue`.

## API surface

Base router:

- Direct service path: `/api/v1/auth`
- Development docs path: `/api/v1/auth/docs`
- Current Kubernetes gateway path: `/api/v1/auth`

### Endpoints

| Method | Path | Purpose | Auth | Permission |
| --- | --- | --- | --- | --- |
| GET | `/api/v1/auth/health` | Health check | No | None |
| POST | `/api/v1/auth/signup` | Create organization, user, OWNER assignment, token | No | None |
| POST | `/api/v1/auth/login` | Validate credentials and issue token | No | None |

### Request and response models

`POST /api/v1/auth/signup`

- Request fields:
  - `organization_name`: 2-150 chars
  - `organization_slug`: 2-100 chars, lowercase letters, digits, hyphen
  - `email`: valid email
  - `password`: minimum 8 chars
- Response fields:
  - `access_token`
  - `token_type`
  - `user_id`
  - `email`
  - `organization_name`

`POST /api/v1/auth/login`

- Request fields:
  - `organization_slug`
  - `email`
  - `password`
- Response fields:
  - `access_token`
  - `token_type`

## Data model

### Core tables

- `organizations`: `id`, `name`, `slug`, `created_at`
- `users`: `id`, `email`, `password_hash`, `is_active`, `is_verified`, `created_at`
- `organization_users`: organization to user membership join table
- `roles`: role catalog
- `permissions`: permission catalog
- `user_roles`: role assignment per organization membership
- `role_permissions`: permission mapping per role
- `refresh_tokens`: token persistence model exists, but refresh-token APIs are not implemented in the current router

## Roles and permissions

### Seeded roles

- `OWNER`
- `ADMIN`
- `SALES`
- `ACCOUNTANT`

### Seeded permissions

- `customer.create`
- `customer.read`
- `customer.update`
- `customer.delete`
- `order.create`
- `order.read`
- `order.update`
- `order.confirm`
- `order.cancel`
- `invoice.create`
- `invoice.read`
- `invoice.update`
- `invoice.cancel`
- `payment.create`
- `payment.read`
- `payment.refund`

### Current assignment behavior

- New signup creates an organization and user.
- The new user is linked into `organization_users`.
- The service assigns the `OWNER` role to that membership.
- The token includes the effective permission list computed from role mappings.

## Communication with other services

### Outbound

- Notification service
  - Transport: RabbitMQ via Celery
  - Task name: `notification.send_signup_email`
  - Payload includes `user_id`, `email`, and `organization_name`

### Inbound dependencies

- None for request-time service-to-service HTTP calls.
- Depends on shared MySQL.
- Depends on RabbitMQ for async notifications.

## Security model

- JWT algorithm: `HS256`
- Secret source: `JWT_SECRET_KEY`
- Expiry source: `ACCESS_TOKEN_EXPIRE_MINUTES`
- Token claims created by the service:
  - `user_id`
  - `email`
  - `org_id`
  - `permissions`
  - `exp`

## Deployment config and secrets

- Secret name: `auth-secret`
- Required secret keys:
  - `DATABASE_URL`
  - `JWT_SECRET_KEY`
  - `ENVIRONMENT`
  - `ACCESS_TOKEN_EXPIRE_MINUTES`
  - `RABBITMQ_URL`
- Required ConfigMap keys: None

## Dockerfile

The Auth Service uses a multi-stage Python Docker build.

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

## Operational notes

- API docs are disabled when `ENVIRONMENT=production`.
- All Auth Service endpoints use the `/api/v1` prefix.
- Request ID middleware propagates `X-Request-ID` on responses.
- A `notification_tasks.py` stub exists locally so task names are registered, but the actual email work happens in `sales-notification-service/`.

## GitOps workflow

If you want to learn more about the GitOps workflow documentation of this microservice, go to this link: `<gitops-doc-link-for-auth-service>`
