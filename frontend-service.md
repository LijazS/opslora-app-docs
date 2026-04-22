# Frontend Service

## Overview

The frontend service is a Next.js application that provides the user-facing Opslora experience for authentication, dashboard analytics, customer management, order management, invoice management, and payment entry.

## Purpose

- Provide the browser UI for the sales workflow
- Store and forward the JWT access token
- Call backend APIs through a shared client
- Gate dashboard pages behind authentication
- Orchestrate cross-service user journeys such as order-to-invoice and invoice-to-payment

## Location

- Source: `sales-frontend-ui/`
- Package manifest: `sales-frontend-ui/package.json`
- API client: `sales-frontend-ui/lib/api.ts`
- Auth provider: `sales-frontend-ui/lib/auth-context.tsx`
- App shell: `sales-frontend-ui/app/(dashboard)/layout.tsx`
- Docker build: `sales-frontend-ui/Dockerfile`
- Local compose service: `docker-compose.yaml`
- Kubernetes deployment: `k8s/Deployments/frontend-deployment.yaml`
- Kubernetes service: `k8s/Services/frontend-service.yaml`
- Kubernetes route: `k8s/httproutes/frontend-route.yaml`

## Stack

- Next.js `16.1.3`
- React `19.2.3`
- TypeScript
- Tailwind CSS 4
- Radix UI primitives
- Framer Motion
- Recharts
- Sonner

## Authentication behavior

- Login and signup pages are public.
- Dashboard routes are protected by `sales-frontend-ui/app/(dashboard)/layout.tsx`.
- JWT token is stored in `localStorage` under `token`.
- `apiFetch(...)` automatically sends `Authorization: Bearer <token>`.
- A `401` response clears the stored token and redirects the browser to `/auth/login`.

## Application pages

| Route | File | Purpose |
| --- | --- | --- |
| `/auth/login` | `app/auth/login/page.tsx` | Login screen |
| `/auth/signup` | `app/auth/signup/page.tsx` | Organization signup screen |
| `/` | `app/(dashboard)/page.tsx` | Dashboard metrics and charts |
| `/customers` | `app/(dashboard)/customers/page.tsx` | Customer list, create, update |
| `/orders` | `app/(dashboard)/orders/page.tsx` | Order list, filters, create, update, confirm, cancel |
| `/orders/[id]/create-invoice` | `app/(dashboard)/orders/[id]/create-invoice/page.tsx` | Invoice creation flow from order |
| `/invoices` | `app/(dashboard)/invoices/page.tsx` | Invoice list, cancel, refund, payment navigation |
| `/invoices/[id]` | `app/(dashboard)/invoices/[id]/page.tsx` | Invoice detail with payment history |
| `/invoices/[id]/pay` | `app/(dashboard)/invoices/[id]/pay/page.tsx` | Add payment flow |

## Features

### Auth flows

- Signup auto-generates a slug from organization name, with manual override
- Login requires organization slug, email, and password

### Dashboard

- Loads invoices, orders, and customers in parallel
- Shows total revenue, total customers, total orders, average invoice value
- Includes a line chart for revenue over time
- Includes a pie chart for invoice status distribution

### Customers

- Paginated customer listing
- Create customer dialog
- Edit customer dialog

### Orders

- Paginated listing
- Filters by status and customer
- Order detail modal
- Create and edit order dialogs
- Confirm and cancel actions
- Shortcut to create an invoice for confirmed orders without an existing invoice

### Invoices

- Paginated listing
- Cancel action for unpaid invoices
- Refund action for paid invoices
- Detail view joins invoice, order, customer, and payment history data

### Payments

- Displays already-paid amount and remaining amount
- Adds payments by method
- Redirects back to invoice detail after success

## Backend APIs used by the frontend

Shared API prefix in code:

- `sales-frontend-ui/lib/api.ts` currently prefixes requests with `/api/v1`

Routes used by the UI:

- Auth:
- `POST /auth/signup`
- `POST /auth/login`
- Customers:
- `GET /customers`
- `POST /customers/create-customer`
- `GET /customers/{id}`
- `PUT /customers/{id}`
- Orders:
- `GET /orders`
- `GET /orders/{id}`
- `POST /orders/create-order/`
- `PUT /orders/{id}`
- `POST /orders/{id}/confirm`
- `POST /orders/{id}/cancel`
- Invoices:
- `GET /invoices`
- `GET /invoices/{id}`
- `POST /invoices/orders/{id}`
- `POST /invoices/{id}/cancel`
- Payments:
- `GET /payments/invoice/{id}`
- `POST /payments/pay`
- `POST /payments/refund/{id}`

## Communication with other services

- Frontend -> Auth service for signup and login
- Frontend -> Customer service for customer management
- Frontend -> Order service for order management
- Frontend -> Invoice service for invoice management
- Frontend -> Payment service for payment and refund flows

The frontend does not communicate directly with MySQL or RabbitMQ.

## Deployment config and secrets

Required secret keys:

- None

Required ConfigMap keys:

- None

## Roles and permissions

- The frontend only checks whether a token exists.
- Permission enforcement is server-side in the backend services.
- UI behavior currently assumes the caller has the necessary permissions for the screens they access.

## Current integration note

The frontend API client uses `/api/v1` for every request, but the current backend route shapes and Kubernetes `HTTPRoute` manifests are not completely consistent, especially for invoice and payment traffic. If you publish these docs for operators, include a routing verification step.

## GitOps workflow

If you want to learn more about the GitOps workflow documentation of this microservice, go to this link: `<gitops-doc-link-for-frontend-service>`
