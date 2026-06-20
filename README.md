# Inventory Management API

Production-ready REST API built with **Node.js**, **Express**, **PostgreSQL**, and **Better Auth**.

## Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js 20 |
| Framework | Express.js |
| Database | PostgreSQL 16 |
| Auth | Better Auth |
| Validation | express-validator |
| Logging | Winston |
| Containerization | Docker + Compose |

---

## Folder Structure

```
inventory-api/
├── src/
│   ├── config/
│   │   ├── database.js        # pg Pool + query helpers
│   │   └── auth.js            # Better Auth instance
│   ├── controllers/           # Thin — delegates to services
│   │   ├── category.controller.js
│   │   ├── supplier.controller.js
│   │   ├── product.controller.js
│   │   ├── inventory.controller.js
│   │   └── dashboard.controller.js
│   ├── services/              # Business logic + transactions
│   │   ├── category.service.js
│   │   ├── supplier.service.js
│   │   ├── product.service.js
│   │   ├── inventory.service.js
│   │   └── dashboard.service.js
│   ├── models/                # DB queries
│   │   ├── category.model.js
│   │   ├── supplier.model.js
│   │   ├── product.model.js
│   │   ├── inventory.model.js
│   │   └── dashboard.model.js
│   ├── middleware/
│   │   ├── authenticate.js    # Better Auth session verification
│   │   ├── authorize.js       # Role-based authorization
│   │   ├── validate.js        # express-validator result handler
│   │   └── errorHandler.js    # Central error handler + 404
│   ├── routes/
│   │   └── index.js           # All routes wired here
│   ├── validators/
│   │   └── index.js           # All express-validator chains
│   ├── utils/
│   │   ├── logger.js          # Winston logger
│   │   ├── response.js        # Consistent API response helpers
│   │   └── errors.js          # Custom error classes
│   ├── app.js                 # Express app setup
│   └── server.js              # Entry point + graceful shutdown
├── migrations/
│   └── 001_initial.sql        # Full schema + seed data
├── scripts/
│   └── migrate.js             # Migration runner
├── logs/                      # Runtime logs (gitignored)
├── Dockerfile
├── docker-compose.yml
├── .env.example
└── package.json
```

---

## Quick Start

### 1. Environment Setup

```bash
cp .env.example .env
# Edit .env with your values
```

### 2. Docker (recommended)

```bash
docker-compose up --build
```

This will:
- Start PostgreSQL
- Run database migrations automatically
- Start the API on port 3000

### 3. Manual Setup

```bash
npm install
# Ensure PostgreSQL is running, then:
npm run migrate
npm run dev
```

---

## Authentication

Better Auth handles auth at `/api/auth/**`.

### Sign Up
```http
POST /api/auth/sign-up/email
Content-Type: application/json

{ "name": "John Doe", "email": "john@example.com", "password": "SecurePass123" }
```

### Sign In
```http
POST /api/auth/sign-in/email
Content-Type: application/json

{ "email": "john@example.com", "password": "SecurePass123" }
```

### Get Session
```http
GET /api/auth/session
Cookie: better-auth.session_token=...
```

### Sign Out
```http
POST /api/auth/sign-out
```

---

## Authorization Roles

| Role | Permissions |
|------|------------|
| `admin` | Full access including deletes and user management |
| `manager` | Create, read, update on all resources; no deletes |
| `staff` | Read access + stock in/out operations |

---

## API Endpoints

Base URL: `http://localhost:3000/api/v1`

### Categories
| Method | Path | Auth |
|--------|------|------|
| GET | `/categories` | any |
| GET | `/categories/:id` | any |
| POST | `/categories` | manager+ |
| PUT | `/categories/:id` | manager+ |
| DELETE | `/categories/:id` | admin |

### Suppliers
| Method | Path | Auth |
|--------|------|------|
| GET | `/suppliers` | any |
| GET | `/suppliers/:id` | any |
| POST | `/suppliers` | manager+ |
| PUT | `/suppliers/:id` | manager+ |
| DELETE | `/suppliers/:id` | admin |

### Products
| Method | Path | Auth |
|--------|------|------|
| GET | `/products` | any |
| GET | `/products/low-stock` | any |
| GET | `/products/:id` | any |
| POST | `/products` | manager+ |
| PUT | `/products/:id` | manager+ |
| DELETE | `/products/:id` | admin |

Query params: `?search=`, `?categoryId=`, `?supplierId=`, `?lowStock=true`, `?page=`, `?limit=`

### Inventory
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/inventory/stock-in` | Add stock | staff+ |
| POST | `/inventory/stock-out` | Remove stock | staff+ |
| POST | `/inventory/adjust` | Manual adjustment | manager+ |
| GET | `/inventory/transactions` | Transaction history | any |
| GET | `/inventory/transactions/:id` | Single transaction | any |
| GET | `/inventory/summary/:productId` | Product summary | any |

### Dashboard
| Method | Path | Auth |
|--------|------|------|
| GET | `/dashboard` | manager+ |
| GET | `/dashboard/overview` | any |
| GET | `/dashboard/stock-movement?days=30` | any |

### Users (Admin only)
| Method | Path |
|--------|------|
| GET | `/users` |
| PATCH | `/users/:id/role` |
| PATCH | `/users/:id/status` |

---

## Response Format

```json
{
  "success": true,
  "message": "Products retrieved",
  "data": { ... },
  "meta": { "pagination": { ... } }
}
```

Error:
```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [{ "field": "sku", "message": "SKU is required" }]
}
```

---

## Stock In Example

```http
POST /api/v1/inventory/stock-in
Authorization: Bearer <session-token>
Content-Type: application/json

{
  "productId": "uuid-here",
  "quantity": 50,
  "unitCost": 12.50,
  "reference": "PO-2024-001",
  "supplierId": "uuid-here",
  "notes": "Initial stock"
}
```

---

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | Server port | `3000` |
| `NODE_ENV` | Environment | `development` |
| `DB_HOST` | PostgreSQL host | `localhost` |
| `DB_PORT` | PostgreSQL port | `5432` |
| `DB_NAME` | Database name | — |
| `DB_USER` | Database user | — |
| `DB_PASSWORD` | Database password | — |
| `BETTER_AUTH_SECRET` | Auth secret (32+ chars) | — |
| `BETTER_AUTH_URL` | Base URL for auth | — |
| `RATE_LIMIT_MAX` | Max requests per window | `100` |
| `LOG_LEVEL` | Winston log level | `info` |
