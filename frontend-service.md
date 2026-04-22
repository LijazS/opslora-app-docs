# Frontend Service

## Overview

The frontend service is a Next.js application that provides the user-facing Opslora experience for authentication, dashboard analytics, customer management, order management, invoice management, and payment entry.

## Purpose

- Provide the browser UI for the sales workflow
- Store and forward the JWT access token
- Call backend APIs through a shared client
- Gate dashboard pages behind authentication
- Orchestrate cross-service user journeys such as order-to-invoice and invoice-to-payment

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

- Shared API prefix in code:
  - `sales-frontend-ui/lib/api.ts` currently prefixes requests with `/api/v1`
- Routes used by the UI:
  - Auth:
    - `POST /api/v1/auth/signup`
    - `POST /api/v1/auth/login`
  - Customers:
    - `GET /api/v1/customers`
    - `POST /api/v1/customers/create-customer`
    - `GET /api/v1/customers/{id}`
    - `PUT /api/v1/customers/{id}`
  - Orders:
    - `GET /api/v1/orders`
    - `GET /api/v1/orders/{id}`
    - `POST /api/v1/orders/create-order/`
    - `PUT /api/v1/orders/{id}`
    - `POST /api/v1/orders/{id}/confirm`
    - `POST /api/v1/orders/{id}/cancel`
  - Invoices:
    - `GET /api/v1/invoices`
    - `GET /api/v1/invoices/{id}`
    - `POST /api/v1/invoices/orders/{id}`
    - `POST /api/v1/invoices/{id}/cancel`
  - Payments:
    - `GET /api/v1/payments/invoice/{id}`
    - `POST /api/v1/payments/pay`
    - `POST /api/v1/payments/refund/{id}`

## Communication with other services

- Frontend to Auth service for signup and login
- Frontend to Customer service for customer management
- Frontend to Order service for order management
- Frontend to Invoice service for invoice management
- Frontend to Payment service for payment and refund flows

The frontend does not communicate directly with MySQL or RabbitMQ.

## Deployment config and secrets

- Required secret keys: None
- Required ConfigMap keys: None

## Dockerfile

The Frontend Service uses a multi-stage Node.js Docker build.

Security approach:

- The runtime image uses `dhi.io/node:20-debian13`, which is a hardened base image intended for safer production deployments.
- Hardened images help reduce unnecessary packages and lower the attack surface of the container.
- The multi-stage build separates build-time tooling from the final runtime image so only the assets needed to serve the app are shipped.
- Production dependencies are installed separately with `npm ci --omit=dev`, which avoids carrying development packages into runtime.
- The container runs as the non-root `node` user, which reduces the blast radius if the application or container is compromised.
- Controlled ownership during `COPY` helps ensure the runtime user has the required access without granting broader privileges.

Build stages:

- Builder stage:
  - Starts from `node:20-alpine`
  - Copies `package*.json`
  - Runs `npm ci`
  - Copies the full application source
  - Builds the production Next.js output with `npm run build`
- Dependency stage:
  - Starts from `node:20-alpine`
  - Installs production-only dependencies with `npm ci --omit=dev`
- Runtime stage:
  - Starts from `dhi.io/node:20-debian13`
  - Copies production `node_modules` from the `deps` stage
  - Copies `.next`, `public`, `package.json`, and `next.config.*` from the `builder` stage

Runtime configuration:

- Working directory: `/app`
- `NODE_ENV=production`
- Runs as the non-root `node` user
- Exposes port `3000`
- Starts the app with `node node_modules/next/dist/bin/next start`

## Roles and permissions

- The frontend only checks whether a token exists.
- Permission enforcement is server-side in the backend services.
- UI behavior currently assumes the caller has the necessary permissions for the screens they access.

## Current integration note

The frontend API client uses `/api/v1` for every backend request.

## GitOps workflow

If you want to learn more about the GitOps workflow documentation of this microservice, go to this link: `<gitops-doc-link-for-frontend-service>`
