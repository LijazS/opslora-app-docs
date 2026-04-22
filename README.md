# Opslora Sales Platform Documentation


## Documentation Map

- [Platform Overview](#platform-overview)
- [Auth Service](./auth-service.md)
- [Customer Service](./customer-service.md)
- [Order Service](./order-service.md)
- [Invoice Service](./invoice-service.md)
- [Payment Service](./payment-service.md)
- [Notification Service](./notification-service.md)
- [Frontend Service](./frontend-service.md)

## Platform Overview

![Application Diagram](./images/application_diagram.jpg)

The platform is a multi-service sales workflow built around one frontend, five synchronous business APIs, one asynchronous notification worker, MySQL, RabbitMQ, and Kubernetes-based service routing.

### High-level request flow

1. A user opens the Next.js frontend and authenticates through the auth service.
2. The auth service validates the organization slug, email, and password, then issues a JWT containing `user_id`, `org_id`, and a `permissions` array.
3. The frontend stores that token in `localStorage` and sends it on later API calls as `Authorization: Bearer <token>`.
4. Each backend service decodes the JWT locally with the shared `JWT_SECRET_KEY`; they do not call the auth service for token introspection.
5. Business services enforce permissions through route dependencies such as `require_permission("order.create")`.
6. When one service depends on another, it forwards the same authorization header so downstream services can preserve tenant and permission context.
7. MySQL stores the system-of-record data for auth, customers, orders, invoices, and payments.
8. RabbitMQ carries async events for email delivery. The auth and order services publish notification tasks, and the notification service consumes them with Celery.

### Current service communication flow

- `Frontend -> Auth Service`: signup and login.
- `Frontend -> Customer Service`: customer CRUD.
- `Frontend -> Order Service`: order create, list, update, confirm, cancel.
- `Frontend -> Invoice Service`: invoice creation, listing, lookup, cancel, status read.
- `Frontend -> Payment Service`: add payment, fetch payments for an invoice, refund.
- `Order Service -> Customer Service`: fetches customer details before creating an order.
- `Invoice Service -> Order Service`: fetches order details before creating an invoice.
- `Payment Service -> Invoice Service`: fetches invoice details before taking payment and updates invoice status after payment or refund.
- `Auth Service -> Notification Service`: publishes a signup email task via RabbitMQ.
- `Order Service -> Notification Service`: publishes order created, confirmed, and cancelled email tasks via RabbitMQ.

### Shared architectural patterns

- Multi-tenancy: business records are scoped by `organization_id`.
- Local JWT validation: every protected service decodes the token independently.
- RBAC: auth seeds the permission catalog; downstream services enforce permissions from JWT claims.
- Request tracing: auth, customer, and order include request ID middleware and forward `X-Request-ID` where implemented.
- Docs toggle: backend OpenAPI docs are disabled when `ENVIRONMENT=production`.

### Runtime components

- Frontend: `sales-frontend-ui/`
- Database: MySQL 8.4 via `docker-compose.yaml`
- Message broker: RabbitMQ via `docker-compose.yaml`
- Kubernetes manifests: `k8s/`
- Helm app charts reference: `opslora-helm-charts-ref/opslora-helm/apps/`

### Routing note

The service source code, frontend client, and Kubernetes `HTTPRoute` manifests are not fully aligned on external URL prefixes:

- Auth and customer/order services mount routers under `/api/v1/...`.
- Invoice and payment services mount routers directly under `/invoices` and `/payments`.
- `sales-frontend-ui/lib/api.ts` currently prefixes all frontend API calls with `/api/v1`.
- The Kubernetes `HTTPRoute` objects use `/api/auth`, `/api/customers`, `/api/orders`, `/api/invoices`, and `/api/payment`.

The service documents below call out direct service routes and current Kubernetes exposure separately where that distinction matters.

### Configuration and secrets source of truth

The documentation in this folder now uses two deployment references:

- Application code in this workspace for what each service actually reads at runtime
- Helm values under `opslora-helm-charts-ref/opslora-helm/apps/<service>/values*.yaml` for the keys currently provisioned into Kubernetes

Where the Helm chart injects a key that the current service code does not appear to read, the service documents call that out explicitly so the docs do not overstate runtime requirements.
