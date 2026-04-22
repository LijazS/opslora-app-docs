# Opslora Sales Platform

Opslora is a microservices-based sales management application that supports the complete business workflow from organization onboarding and user authentication to customer management, order processing, invoicing, payments, and notifications. The platform is designed as a set of focused services that work together to provide a scalable and maintainable sales operations system.

## Application Diagram

![Application Diagram](./images/application_diagram.svg)

The application is built around independent microservices, each responsible for a specific domain within the sales lifecycle. Together, these services power the frontend experience, core business processes, financial operations, and background communication workflows.

## Microservices Overview

### Auth Service

The auth service manages organization onboarding, user authentication, JWT generation, and role-based access control. It acts as the identity and access management layer of the platform.

For more information, refer to [auth-service.md](./auth-service.md).

### Customer Service

The customer service manages customer information for each organization and serves as the central source of customer records used across the platform.

For more information, refer to [customer-service.md](./customer-service.md).

### Order Service

The order service handles the creation and lifecycle management of sales orders, including order items and status transitions throughout the sales process.

For more information, refer to [order-service.md](./order-service.md).

### Invoice Service

The invoice service creates and manages invoices for confirmed orders, supporting invoice tracking and the financial flow of the application.

For more information, refer to [invoice-service.md](./invoice-service.md).

### Payment Service

The payment service records payments against invoices and supports the tracking of outstanding and completed payment activity.

For more information, refer to [payment-service.md](./payment-service.md).

### Notification Service

The notification service handles asynchronous communication such as transactional emails for signup and order-related events.

For more information, refer to [notification-service.md](./notification-service.md).

### Frontend Service

The frontend service is the user-facing application that provides the dashboards and workflows used to interact with the platform and its business features.

For more information, refer to [frontend-service.md](./frontend-service.md).
