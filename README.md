# Ticket Booking App (Microservices)

A production-style ticket marketplace built with **Node.js/TypeScript microservices**, **event-driven communication**, and a **Next.js** client. Users can sign up/sign in, create (sell) tickets, purchase tickets by creating orders, and pay via Stripe before the order expires.

## Key features

- **Authentication**: Sign up / sign in / sign out with JWT stored in cookie sessions
- **Tickets**: Create, list, view, and update tickets (updates blocked while reserved)
- **Orders**: Create orders to reserve tickets, view your orders, cancel orders
- **Order expiration**: Orders automatically expire after ~15 minutes if unpaid
- **Payments**: Stripe-based payments that complete orders
- **Reservation guarantees**: Prevents purchasing a ticket that is already reserved

## Architecture (services)

- **`client`**: Next.js UI (React)
- **`auth`**: User auth + current-user
- **`tickets`**: Ticket creation/listing/updating and reservation state
- **`orders`**: Order creation/cancellation + order state lifecycle
- **`payments`**: Stripe charge creation + payment records
- **`expiration`**: Delayed job scheduler that emits expiration events
- **`common`**: Shared library for middleware, typed events, and error handling

Event bus + async processing:

- **NATS Streaming** for domain events (ticket created/updated, order created/cancelled, payment created, expiration complete)
- **Bull + Redis** for delayed expiration jobs

## Tech stack

**Backend**
- Node.js + **TypeScript**
- **Express**, `express-validator`, `express-async-errors`
- **MongoDB** + **Mongoose** (with optimistic concurrency via `mongoose-update-if-current`)
- **NATS Streaming** (`node-nats-streaming`)
- **Bull** + **Redis**
- **Stripe**

**Frontend**
- **Next.js** + React
- Axios
- Bootstrap
- `react-stripe-checkout`

**Testing**
- Jest + Supertest
- `mongodb-memory-server`

**Infrastructure / Dev**
- Docker
- Kubernetes
- Skaffold
- NGINX Ingress

## Local development (Kubernetes + Skaffold)

This repo is designed to run locally on Kubernetes with an ingress host of **`ticketing.dev`**.

### Prerequisites

- Docker Desktop with Kubernetes enabled (or any local K8s cluster)
- `kubectl`
- `skaffold`
- NGINX ingress controller installed in your cluster

### 1) Point the domain to localhost

Add this entry to your hosts file:

- Windows: `C:\Windows\System32\drivers\etc\hosts`

```
127.0.0.1 ticketing.dev
```

### 2) Create Kubernetes secrets

The services require the following secrets:

- `jwt-secret` (JWT signing key)
- `stripe-secret` (Stripe secret key used by the payments service)

Create them:

```bash
kubectl create secret generic jwt-secret --from-literal=JWT_KEY=your_jwt_secret
kubectl create secret generic stripe-secret --from-literal=STRIPE_KEY=your_stripe_secret
```

> Tip: use a Stripe **test** secret key (starts with `sk_test_...`) for local development.

### 3) Start everything with Skaffold

From the directory containing `skaffold.yaml`:

```bash
skaffold dev
```

This will:
- apply Kubernetes manifests from `infra/k8s/*`
- build and deploy all services
- sync source changes into running containers

### 4) Open the app

Visit:

- `http://ticketing.dev`

## High-level API routes

Ingress routes are mounted under `ticketing.dev`:

- **Auth**: `/api/users/*`
  - `POST /api/users/signup`
  - `POST /api/users/signin`
  - `POST /api/users/signout`
  - `GET  /api/users/currentuser`
- **Tickets**: `/api/tickets/*`
  - `GET  /api/tickets`
  - `GET  /api/tickets/:id`
  - `POST /api/tickets`
  - `PUT  /api/tickets/:id`
- **Orders**: `/api/orders/*`
  - `GET    /api/orders`
  - `GET    /api/orders/:orderId`
  - `POST   /api/orders`
  - `DELETE /api/orders/:orderId`
- **Payments**: `/api/payments/*`
  - `POST /api/payments`

## Notes

- **Ticket listing excludes reserved tickets** (tickets with an `orderId` are not returned from the tickets index route).
- The UI uses a Stripe **publishable test key** for checkout; the server uses `STRIPE_KEY` from `stripe-secret`.

## Repo structure

```
client/       Next.js frontend
auth/         Auth service
tickets/      Tickets service
orders/       Orders service
payments/     Payments service
expiration/   Expiration worker (Bull/Redis)
common/       Shared library package
infra/k8s/    Kubernetes manifests (including ingress + NATS)
skaffold.yaml Local dev orchestration
```

