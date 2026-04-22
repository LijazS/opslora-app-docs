# Invoice Service

## Overview

The invoice service creates one invoice per confirmed order, computes tax and discounts, tracks invoice status, and exposes the invoice data consumed by both the frontend and the payment service.

## Purpose

- Create an invoice from a confirmed order
- Enforce one invoice per order
- Calculate subtotal, tax, discount, total, and due date
- List and fetch invoices
- Cancel unpaid invoices
- Accept status changes from payment workflows

## Features

- Invoice creation from orders
- Enforced order confirmation prerequisite
- 18 percent tax calculation
- Support for `FLAT` and `PERCENT` discounts in the service layer
- Due date default of 30 days from creation
- Invoice status transitions driven by invoice and payment actions

## API surface

Base router:

- Direct service path: `/api/v1/invoices`
- Development docs path: `/api/v1/invoices/docs`
- Current Kubernetes gateway path: `/api/v1/invoices`

### Endpoints

| Method | Path | Purpose | Auth | Permission |
| --- | --- | --- | --- | --- |
| GET | `/api/v1/invoices/health` | Health check | No | None |
| POST | `/api/v1/invoices/orders/{order_id}` | Create invoice from order | Yes | `invoice.create` |
| GET | `/api/v1/invoices/{invoice_id}` | Fetch one invoice | Yes | `invoice.read` |
| GET | `/api/v1/invoices/` | List invoices | Yes | `invoice.read` |
| POST | `/api/v1/invoices/{invoice_id}/cancel` | Cancel invoice | Yes | `invoice.cancel` |
| POST | `/api/v1/invoices/{invoice_id}/status` | Update invoice status | Yes | `invoice.update` |

### Query parameters

`GET /api/v1/invoices/`

- `status`: optional
- `order_id`: optional

### Request and response models

- Create endpoint:
  - Path parameter: `order_id`
  - The frontend currently sends discount fields, but the current router does not bind them from the request body.
  - The service function supports:
    - `discount_type`
    - `discount_value`
  - In the current API implementation those arguments stay at their defaults unless the route is expanded.
- Invoice response:
  - `id`
  - `order_id`
  - `subtotal`
  - `tax`
  - `total`
  - `due_date`
  - `status`
  - `discount_type`
  - `discount_value`
  - `created_at`
- Status update payload:
  - `status`
  - Allowed values:
    - `UNPAID`
    - `PARTIALLY_PAID`
    - `PAID`
    - `OVERDUE`
    - `CANCELLED`
    - `REFUNDED`

## Data model

`invoices`

- `id`
- `organization_id`
- `order_id` (unique)
- `subtotal`
- `tax`
- `total`
- `discount_type`
- `discount_value`
- `due_date`
- `status`
- `created_by_user_id`
- `created_at`

### Current constraints

- One invoice per order
- Status check constraint on allowed values
- Discount type check constraint for `FLAT` and `PERCENT`

## Status rules

- New invoices start as `UNPAID`.
- Invoice creation is allowed only for `CONFIRMED` orders.
- Only `UNPAID` invoices can be cancelled.
- Payment service updates invoice status to `PARTIALLY_PAID`, `PAID`, or `REFUNDED`.

## Roles and permissions

- `invoice.create`
- `invoice.read`
- `invoice.update`
- `invoice.cancel`

## Communication with other services

### Outbound synchronous HTTP

- Order service
  - Environment variable: `ORDER_SERVICE_URL`
  - Request used: `GET {ORDER_SERVICE_URL}/api/v1/orders/{order_id}`
  - Forwarded headers:
    - `Authorization`
  - Purpose:
    - Validate order existence
    - Check order status is `CONFIRMED`
    - Pull line items for subtotal and tax calculation

### Inbound synchronous HTTP

- Frontend service for invoice UI flows
- Payment service for invoice reads and invoice status updates

## Deployment config and secrets

- Secret name: `invoice-secret`
- Required secret keys:
  - `DATABASE_URL`
  - `JWT_SECRET_KEY`
  - `ENVIRONMENT`
- ConfigMap name: `invoice-config`
- Required ConfigMap keys:
  - `ORDER_SERVICE_URL`

## Dockerfile

The Invoice Service uses a multi-stage Python Docker build.

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
- All Invoice Service endpoints use the `/api/v1` prefix.
- No request ID middleware is currently configured here.
- Overdue status exists in the enum and database constraint, but no background job currently auto-transitions invoices to `OVERDUE`.

## GitOps workflow

If you want to learn more about the GitOps workflow documentation of this microservice, go to this link: `<gitops-doc-link-for-invoice-service>`
