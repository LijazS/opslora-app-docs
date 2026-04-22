# Payment Service

## Overview

The payment service records payments against invoices, calculates whether an invoice is partially paid or fully paid, and drives refund status changes back into the invoice service.

## Purpose

- Record a payment for an invoice
- Prevent overpayment
- List payments for an invoice
- Mark fully paid invoices as refunded
- Synchronize invoice status with the invoice service

## Location

- Source: `sales-payment-service/`
- FastAPI entrypoint: `sales-payment-service/app/main.py`
- Router: `sales-payment-service/app/routers/payments.py`
- Business logic: `sales-payment-service/app/services/payment_service.py`
- Service client: `sales-payment-service/app/utils/service_client.py`
- Model: `sales-payment-service/app/models/payment.py`
- Docker build: `sales-payment-service/Dockerfile`
- Local compose service: `docker-compose.yaml`
- Kubernetes deployment: `k8s/Deployments/payment-deployment.yaml`
- Kubernetes service: `k8s/Services/payment-service.yaml`
- Kubernetes route: `k8s/httproutes/payment-route.yaml`

## Features

- Payment creation with multiple payment methods
- Partial payment support
- Overpayment prevention
- Invoice status synchronization
- Refund flow for fully paid invoices

## API surface

Base router:

- Direct service path: `/payments`
- Development docs path: `/payments/docs`
- Frontend API client currently calls payment APIs through `/api/v1/payments/...`
- Current Kubernetes gateway path in `k8s/httproutes/payment-route.yaml`: `/api/payment`

The route-prefix mismatch above is important to keep in mind when using or publishing these docs.

### Endpoints

| Method | Path | Purpose | Auth | Permission |
| --- | --- | --- | --- | --- |
| GET | `/payments/health` | Health check | No | None |
| POST | `/payments/pay` | Record a payment | Yes | `payment.create` |
| GET | `/payments/invoice/{invoice_id}` | List payments for an invoice | Yes | `payment.read` |
| POST | `/payments/refund/{invoice_id}` | Mark invoice as refunded | Yes | `payment.refund` |

### Request and response models

Payment create payload:

- `invoice_id`
- `amount`
- `payment_method`: `CASH`, `CARD`, `UPI`, `BANK_TRANSFER`

Payment response:

- `id`
- `invoice_id`
- `amount`
- `payment_method`
- `paid_at`

Refund response returned by service logic:

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

- Invoice service:
- Environment variable: `INVOICE_SERVICE_URL`
- Reads:
- `GET {INVOICE_SERVICE_URL}/invoices/{invoice_id}`
- Writes:
- `POST {INVOICE_SERVICE_URL}/invoices/{invoice_id}/status`
- Forwarded header:
- `Authorization`

### Inbound synchronous HTTP

- Frontend service for payment entry, payment history, and refund actions

## Deployment config and secrets

Secret name:

- `payment-secret`

Required secret keys:

- `DATABASE_URL`
- `JWT_SECRET_KEY`
- `ENVIRONMENT`

ConfigMap name:

- `payment-config`

Required ConfigMap keys:

- `INVOICE_SERVICE_URL`

## Operational notes

- API docs are disabled in production.
- The refund endpoint changes invoice status but does not create a negative payment entry in the payments table.
- No request ID middleware is currently configured here.

## GitOps workflow

If you want to learn more about the GitOps workflow documentation of this microservice, go to this link: `<gitops-doc-link-for-payment-service>`
