# Payment Service

## Overview

The payment service records payments against invoices, calculates whether an invoice is partially paid or fully paid, and drives refund status changes back into the invoice service.

## Purpose

- Record a payment for an invoice
- Prevent overpayment
- List payments for an invoice
- Mark fully paid invoices as refunded
- Synchronize invoice status with the invoice service

## Features

- Payment creation with multiple payment methods
- Partial payment support
- Overpayment prevention
- Invoice status synchronization
- Refund flow for fully paid invoices

## API surface

Base router:

- Direct service path: `/api/v1/payments`
- Development docs path: `/api/v1/payments/docs`
- Frontend API client currently calls payment APIs through `/api/v1/payments/...`
- Current Kubernetes gateway path in `k8s/httproutes/payment-route.yaml`: `/api/v1/payments`

### Endpoints

| Method | Path | Purpose | Auth | Permission |
| --- | --- | --- | --- | --- |
| GET | `/api/v1/payments/health` | Health check | No | None |
| POST | `/api/v1/payments/pay` | Record a payment | Yes | `payment.create` |
| GET | `/api/v1/payments/invoice/{invoice_id}` | List payments for an invoice | Yes | `payment.read` |
| POST | `/api/v1/payments/refund/{invoice_id}` | Mark invoice as refunded | Yes | `payment.refund` |

### Request and response models

- Payment create payload:
  - `invoice_id`
  - `amount`
  - `payment_method`: `CASH`, `CARD`, `UPI`, `BANK_TRANSFER`
- Payment response:
  - `id`
  - `invoice_id`
  - `amount`
  - `payment_method`
  - `paid_at`
- Refund response returned by service logic:
  - `invoice_id`
  - `status`

## Data model

`payments`

- `id`
- `organization_id`
- `invoice_id`
- `amount`
- `payment_method`
- `created_by_user_id`
- `paid_at`
- `note`

### Current constraints

- `amount > 0`
- `payment_method` limited to `CASH`, `CARD`, `UPI`, `BANK_TRANSFER`
- Payment records reference invoices by ID only; there is no database foreign key to the invoice service table

## Payment and refund rules

- Cancelled invoices cannot be paid.
- Fully paid invoices cannot accept more payments.
- Payment amount must be greater than zero.
- Total paid cannot exceed invoice total.
- If total paid equals invoice total, the invoice is marked `PAID`.
- Otherwise the invoice is marked `PARTIALLY_PAID`.
- Refund is allowed only when invoice status is `PAID` and the paid sum equals the invoice total.
- Refund changes invoice status to `REFUNDED`.

## Roles and permissions

- `payment.create`
- `payment.read`
- `payment.refund`

## Communication with other services

### Outbound synchronous HTTP

- Invoice service
  - Environment variable: `INVOICE_SERVICE_URL`
  - Reads:
    - `GET {INVOICE_SERVICE_URL}/api/v1/invoices/{invoice_id}`
  - Writes:
    - `POST {INVOICE_SERVICE_URL}/api/v1/invoices/{invoice_id}/status`
  - Forwarded header:
    - `Authorization`

### Inbound synchronous HTTP

- Frontend service for payment entry, payment history, and refund actions

## Deployment config and secrets

- Secret name: `payment-secret`
- Required secret keys:
  - `DATABASE_URL`
  - `JWT_SECRET_KEY`
  - `ENVIRONMENT`
- ConfigMap name: `payment-config`
- Required ConfigMap keys:
  - `INVOICE_SERVICE_URL`

## Dockerfile

The Payment Service uses a multi-stage Python Docker build.

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

- API docs are disabled in production.
- All Payment Service endpoints use the `/api/v1` prefix.
- The refund endpoint changes invoice status but does not create a negative payment entry in the payments table.
- No request ID middleware is currently configured here.

## GitOps workflow

If you want to learn more about the GitOps workflow documentation of this microservice, go to this link: `<gitops-doc-link-for-payment-service>`
