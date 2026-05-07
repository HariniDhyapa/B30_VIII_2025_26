# Multi-Tenant E-Commerce Platform (FastAPI + PostgreSQL Schemas)

Production-style, cost-free microservices architecture for multi-tenant e-commerce with strict tenant isolation using PostgreSQL schemas.

## Architecture

- `tenant-user` service: tenant onboarding, seller/customer auth, JWT + OAuth2
- `product` service: product CRUD APIs + Redis caching
- `order` service: cart (Redis), checkout, order history, async updates
- `payment` service: consumes `order.created`, mock processing, emits `payment.succeeded`
- `frontend` service: lightweight browser UI to verify seller/customer flows against the APIs
- Shared infra: PostgreSQL, Redis, RabbitMQ

## Tenant Isolation Model

- Single PostgreSQL instance
- `public.tenants` tracks tenants
- Each tenant gets schema: `tenant_<slug>`
- Business tables (`users`, `products`, `orders`, `payments`) exist per tenant schema
- JWT includes `schema`, every request sets PostgreSQL `search_path` to that schema

## End Users

- Sellers (tenants): small business owners with seller accounts that manage products, orders, and revenue for their own shop
- Customers: shoppers who sign up under a specific tenant storefront and place orders in that tenant's schema
- One backend supports many shops, while JWT claims carry the tenant context needed to keep every shop isolated

## Repository Structure

```text
common/
services/
  tenant-user/
  product/
  order/
  payment/
  frontend/
k8s/
  base/
  minikube/
.github/workflows/
terraform/aws-free-tier/
scripts/
```

## Local Run (Docker Compose)

1. Copy env:

```bash
cp .env.example .env
```

2. Build + run:

```bash
docker compose up --build -d
```

3. Service endpoints:

- Tenant-User: `http://localhost:8001`
- Product: `http://localhost:8002`
- Order: `http://localhost:8003`
- Payment: `http://localhost:8004`
- Storefront URL: `http://localhost:3000/?tenant=<tenant-slug>`
- Platform admin URL: `http://localhost:3000/admin.html`
- RabbitMQ UI: `http://localhost:15672` (`guest/guest`)

### DevOps Improvements Included

- GitHub Actions pipeline with validation, image builds, and smoke testing
- Container health checks in Docker Compose for all app services
- Kubernetes readiness/liveness probes for platform and app workloads
- Basic CPU/memory requests and limits in Kubernetes manifests
- Separate local Docker Compose path and Kubernetes deployment path

## Quick Test Flow (Nike + Adidas isolation)

Use [scripts/smoke_test.ps1](scripts/smoke_test.ps1).

It validates:
- tenant creation for `nike` and `adidas`
- independent JWTs per tenant
- product CRUD isolation (Nike data not visible in Adidas)
- cart -> checkout -> payment event flow

You can also verify the same flows manually in the browser through the frontend test console.

## Client Provisioning Flow

Use [scripts/provision_tenant.ps1](scripts/provision_tenant.ps1) to onboard a new client storefront in one call.

Example:

```powershell
.\scripts\provision_tenant.ps1 -TenantName "Nike" -TenantSlug "nike" -SellerEmail "seller@nike.com"
```

If you set `PROVISIONING_API_KEY` in `.env`, pass the same value with `-ProvisioningApiKey`.

The command returns:
- storefront URL
- admin URL
- seller login email
- tenant schema

Use [scripts/client_handoff_template.md](scripts/client_handoff_template.md) to deliver credentials and operational details to the client.

## Core APIs

### Tenant-User Service
- `POST /tenants/signup`
- `POST /tenants/provision`
- `GET /tenants/{tenant_slug}/settings`
- `PUT /tenants/{tenant_slug}/settings` (requires `x-provision-key` when `PROVISIONING_API_KEY` is set)
- `POST /tenants/{tenant_slug}/seller/reset-password` (requires `x-provision-key` when `PROVISIONING_API_KEY` is set)
- `POST /customers/signup`
- `POST /auth/token`
- `POST /auth/login` (JSON body)
- `POST /auth/token/json` (JSON body)
- `GET /auth/me`
- `POST /platform/auth/login`
- `GET /platform/tenants`
- `POST /platform/tenants/provision`

### Platform Admin Authentication
- Credentials are configured via `.env`:
  - `PLATFORM_ADMIN_EMAIL`
  - `PLATFORM_ADMIN_PASSWORD`
- Platform admin token is required for `GET /platform/tenants`.

### Product Service
- `POST /products`
- `GET /products`
- `GET /products/{id}`
- `PATCH /products/{id}`
- `DELETE /products/{id}`

### Order Service
- `POST /cart/add`
- `DELETE /cart/remove/{product_id}`
- `GET /cart`
- `POST /orders/checkout`
- `GET /orders`

### Payment Service
- `GET /health`
- Background consumer for `order.created`

## Kubernetes (Minikube)

1. Build/push images (or `minikube image load`):

```bash
docker build -f services/tenant-user/Dockerfile -t varshatogari/tenant-user:v1 .
docker build -f services/product/Dockerfile -t varshatogari/product:v1 .
docker build -f services/order/Dockerfile -t varshatogari/order:v1 .
docker build -f services/payment/Dockerfile -t varshatogari/payment:v1 .
docker build -f services/frontend/Dockerfile -t neww-frontend:latest .
```

2. Deploy:

```bash
kubectl apply -k k8s/minikube
```

3. Verify the workloads:

```bash
kubectl get pods -n ecommerce
```

4. Access the services with NodePort or `kubectl port-forward`, for example:

```bash
kubectl port-forward -n ecommerce svc/frontend 3000:3000
kubectl port-forward -n ecommerce svc/tenant-user 8001:8000
kubectl port-forward -n ecommerce svc/product 8002:8000
kubectl port-forward -n ecommerce svc/order 8003:8000
kubectl port-forward -n ecommerce svc/payment 8004:8000
```

### Kubernetes Notes

- Every app service now exposes `/health`, which is used by readiness and liveness probes
- Resource requests/limits are included so the manifests look closer to production-style deployments
- The current setup is intentionally lightweight and keeps your existing Docker workflow untouched


