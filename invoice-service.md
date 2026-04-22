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

- Direct service path: `/invoices`
- Development docs path: `/invoices/docs`
- Current Kubernetes gateway path: `/api/invoices`

### Endpoints

| Method | Path | Purpose | Auth | Permission |
| --- | --- | --- | --- | --- |
| GET | `/invoices/health` | Health check | No | None |
| POST | `/invoices/orders/{order_id}` | Create invoice from order | Yes | `invoice.create` |
| GET | `/invoices/{invoice_id}` | Fetch one invoice | Yes | `invoice.read` |
| GET | `/invoices/` | List invoices | Yes | `invoice.read` |
| POST | `/invoices/{invoice_id}/cancel` | Cancel invoice | Yes | `invoice.cancel` |
| POST | `/invoices/{invoice_id}/status` | Update invoice status | Yes | `invoice.update` |

### Query parameters

`GET /invoices/`

- `status`: optional
- `order_id`: optional

### Request and response models

Create endpoint:

- Path parameter: `order_id`
- The frontend currently sends discount fields, but the current router does not bind them from the request body.
- The service function supports:
- `discount_type`
- `discount_value`
- In the current API implementation those arguments stay at their defaults unless the route is expanded.

Invoice response:

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

Status update payload:

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

- Order service:
- Environment variable: `ORDER_SERVICE_URL`
- Request used: `GET {ORDER_SERVICE_URL}/orders/{order_id}`
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

Secret name:

- `invoice-secret`

Required secret keys:

- `DATABASE_URL`
- `JWT_SECRET_KEY`
- `ENVIRONMENT`

ConfigMap name:

- `invoice-config`

Required ConfigMap keys:

- `ORDER_SERVICE_URL`

## Operational notes

- API docs are disabled in production.
- The service mounts routes without `/api/v1`, so published routes need to be aligned carefully with whichever ingress or gateway layer is used.
- No request ID middleware is currently configured here.
- Overdue status exists in the enum and database constraint, but no background job currently auto-transitions invoices to `OVERDUE`.

## GitOps workflow

If you want to learn more about the GitOps workflow documentation of this microservice, go to this link: `<gitops-doc-link-for-invoice-service>`
