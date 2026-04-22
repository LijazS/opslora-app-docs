# Customer Service

## Overview

The customer service owns customer master data for each organization. It exposes customer CRUD operations and acts as the source of truth for customer validation used by the order service.

## Purpose

- Create customer records.
- List customers for an organization.
- Fetch a customer by ID.
- Update customer details.
- Confirm whether a customer exists within the caller's organization.

## Features

- Organization-scoped customer records
- JWT-based authentication
- Permission-guarded create, read, and update APIs
- Pagination on customer listing
- Email uniqueness enforcement
- Request ID middleware for traceability

## API surface

Base router:

- Direct service path: `/api/v1/customers`
- Development docs path: `/customers/docs`
- Current Kubernetes gateway path: `/api/customers`

### Endpoints

| Method | Path | Purpose | Auth | Permission |
| --- | --- | --- | --- | --- |
| GET | `/api/v1/customers/health` | Health check | No | None |
| POST | `/api/v1/customers/create-customer` | Create a customer | Yes | `customer.create` |
| GET | `/api/v1/customers/` | List customers with `page` and `limit` | Yes | `customer.read` |
| GET | `/api/v1/customers/{customer_id}/exists` | Check customer existence | Yes | JWT only |
| GET | `/api/v1/customers/{customer_id}` | Fetch one customer | Yes | `customer.read` |
| PUT | `/api/v1/customers/{customer_id}` | Update customer | Yes | `customer.update` |

### Query parameters

`GET /api/v1/customers/`

- `page`: default `1`
- `limit`: default `15`, max `100`

### Request and response models

Create and update payloads:

- `name`
- `email`

Customer response:

- `id`
- `name`
- `email`
- `created_at`

## Data model

`customers`

- `id`
- `organization_id`
- `name`
- `email`
- `created_by_user_id`
- `created_at`

### Current data constraints

- `email` is globally unique in the current model, not unique per organization.
- Records are filtered by `organization_id` in service queries.

## Roles and permissions

- `customer.create`: required for creation
- `customer.read`: required for list and detail retrieval
- `customer.update`: required for updates
- `customer.delete`: seeded by auth, but no delete endpoint is implemented in the current service

## Communication with other services

### Inbound

- Frontend service calls this API for customer management pages.
- Order service calls the customer detail endpoint during order creation to validate the customer and capture customer name/email.

### Outbound

- No outbound HTTP or message-broker calls in the current implementation.

## Authentication and authorization

- JWT is decoded locally using `JWT_SECRET_KEY`.
- `get_current_user` extracts `user_id`, `org_id`, and `permissions` from the token.
- `require_permission(...)` enforces RBAC at the route layer.

## Deployment config and secrets

Secret name:

- `customer-secret`

Required secret keys:

- `DATABASE_URL`
- `JWT_SECRET_KEY`
- `ENVIRONMENT`

Required ConfigMap keys:

- None

## Operational notes

- API docs are disabled in production.
- `redirect_slashes=False` is enabled, so trailing slash behavior matters.
- The frontend currently calls the listing endpoint with a trailing slash, which matches the current route shape.
- Request ID middleware adds `X-Request-ID` to responses.

## GitOps workflow

If you want to learn more about the GitOps workflow documentation of this microservice, go to this link: `<gitops-doc-link-for-customer-service>`
