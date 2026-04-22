# Order Service

## Overview

The order service manages sales orders and line items for each organization. It validates the selected customer through the customer service, persists the order, and emits async notification events as the order moves through its lifecycle.

## Purpose

- Create an order for a customer
- Store order line items
- List and filter orders
- Update draft orders
- Confirm orders
- Cancel eligible orders
- Trigger order lifecycle email notifications

## Location

- Source: `sales-order-service/`
- FastAPI entrypoint: `sales-order-service/app/main.py`
- Router: `sales-order-service/app/routers/v1/orders.py`
- Business logic: `sales-order-service/app/services/order_service.py`
- Service client: `sales-order-service/app/utils/service_client.py`
- Models: `sales-order-service/app/models/order.py`, `sales-order-service/app/models/order_item.py`
- Docker build: `sales-order-service/Dockerfile`
- Local compose service: `docker-compose.yaml`
- Kubernetes deployment: `k8s/Deployments/order-deployment.yaml`
- Kubernetes service: `k8s/Services/order-service.yaml`
- Kubernetes route: `k8s/httproutes/order-route.yaml`

## Features

- Order creation with multiple line items
- Customer validation through synchronous HTTP call to customer service
- Status lifecycle control
- Pagination and filtering
- Email notifications for create, confirm, and cancel events
- Request ID propagation to downstream customer calls

## API surface

Base router:

- Direct service path: `/api/v1/orders`
- Development docs path: `/orders/docs`
- Current Kubernetes gateway path: `/api/orders`

### Endpoints

| Method | Path | Purpose | Auth | Permission |
| --- | --- | --- | --- | --- |
| GET | `/api/v1/orders/health` | Health check | No | None |
| POST | `/api/v1/orders/create-order` | Create order | Yes | `order.create` |
| POST | `/api/v1/orders/create-order/` | Create order alternate trailing-slash path | Yes | `order.create` |
| GET | `/api/v1/orders/{order_id}` | Fetch one order with items and computed total | Yes | `order.read` |
| GET | `/api/v1/orders/` | List orders | Yes | `order.read` |
| PUT | `/api/v1/orders/{order_id}` | Replace line items on a CREATED order | Yes | `order.update` |
| POST | `/api/v1/orders/{order_id}/confirm` | Confirm order | Yes | `order.confirm` |
| POST | `/api/v1/orders/{order_id}/cancel` | Cancel order | Yes | `order.cancel` |

### Query parameters

`GET /api/v1/orders/`

- `page`: default `1`
- `limit`: default `15`, max `100`
- `status`: optional
- `customer_id`: optional

### Request and response models

Order create payload:

- `customer_id`
- `items[]`
- Each item has `product_name`, `quantity`, `unit_price`

Order update payload:

- `items[]`

Order response:

- `id`
- `customer_id`
- `status`
- `created_at`
- `total`
- `items[]`

## Data model

### `orders`

- `id`
- `organization_id`
- `customer_id`
- `customer_email`
- `customer_name`
- `status`: `CREATED`, `CONFIRMED`, `CANCELLED`
- `created_by_user_id`
- `created_at`

### `order_items`

- `id`
- `order_id`
- `product_name`
- `quantity`
- `unit_price`

## Status rules

- New orders start as `CREATED`.
- Only `CREATED` orders can be updated.
- Only `CREATED` orders can be confirmed.
- Confirmed orders cannot be cancelled.
- Cancellation is allowed only before confirmation.

## Roles and permissions

- `order.create`
- `order.read`
- `order.update`
- `order.confirm`
- `order.cancel`

## Communication with other services

### Outbound synchronous HTTP

- Customer service:
- Environment variable: `CUSTOMER_SERVICE_URL`
- Request used: `GET {CUSTOMER_SERVICE_URL}{API_VERSION}/customers/{customer_id}`
- Forwarded headers:
- `Authorization`
- `X-Request-ID`
- Purpose:
- Validate that the customer exists in the same organization
- Capture `customer_email` and `customer_name` into the order record

### Outbound asynchronous messaging

- Notification service via RabbitMQ/Celery
- Queue: `notification_queue`
- Tasks:
- `notification.send_order_created_email`
- `notification.send_order_confirmed_email`
- `notification.send_order_cancelled_email`

### Inbound dependencies

- Frontend service for order management screens
- Invoice service for fetching confirmed order details during invoice creation

## Deployment config and secrets

Secret name:

- `order-secret`

Required secret keys:

- `DATABASE_URL`
- `JWT_SECRET_KEY`
- `ENVIRONMENT`
- `RABBITMQ_URL`

ConfigMap name:

- `order-config`

Required ConfigMap keys:

- `CUSTOMER_SERVICE_URL`

## Operational notes

- API docs are disabled in production.
- The service computes order totals dynamically when reading data; totals are not persisted as a dedicated column.
- `API_VERSION` defaults to `/api/v1`.
- The order service stores customer name and email as a snapshot at order creation time.

## GitOps workflow

If you want to learn more about the GitOps workflow documentation of this microservice, go to this link: `<gitops-doc-link-for-order-service>`
