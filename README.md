# ServiceNest — Reliable Local Services, Made Simple

**ServiceNest** is an enterprise-grade, full-stack local service booking web application built with a modern React SPA frontend, an Express/Node.js REST API backend, and a relational SQLite database with strict ACID transactional guarantees.

ServiceNest connects homeowners and businesses with verified local service providers (electricians, plumbers, cleaners, technicians, painters, carpenters, tutors, and salon professionals), featuring real-time availability checking, atomic double-booking prevention, internal role-restricted messaging, post-completion reviews, complaint resolution, and a centralized administrative dashboard.

---

## Table of Contents
1. [Core Features by Role](#1-core-features-by-role)
2. [Technology Stack](#2-technology-stack)
3. [Architecture & Concurrency Protection](#3-architecture--concurrency-protection)
4. [Database Schema](#4-database-schema)
5. [Demo Credentials](#5-demo-credentials)
6. [Quick Start & Installation](#6-quick-start--installation)
7. [Running Tests](#7-running-tests)
8. [API Documentation Overview](#8-api-documentation-overview)
9. [Git & GitHub Setup](#9-git--github-setup)
10. [Production Deployment Guide](#10-production-deployment-guide)
11. [Production-Readiness Checklist](#11-production-readiness-checklist)

---

## 1. Core Features by Role

### A. Customer
- **Discovery & Search**: Full-text and filtered service discovery by keyword, category, city/service area, price range, and minimum provider rating.
- **Dynamic Slot Booking**: Interactive calendar displaying available slots computed against provider weekly working hours, buffer times, blackout dates, and existing appointments.
- **Transparent Price Breakdown**: Itemized calculation of base service price, visit fee, platform fee (5%), and zero hidden taxes.
- **Immutable Price Snapshots**: Agreed pricing is snapshotted into the booking record, safeguarding past receipts from future service price fluctuations.
- **Booking Management & Timeline**: Visual status timeline tracking bookings across `PENDING`, `CONFIRMED`, `IN_PROGRESS`, `COMPLETED`, or `CANCELLED` states.
- **Internal Messaging**: In-app communication channel scoped exclusively between customer and provider for active bookings.
- **Post-Service Reviews**: 1-to-5 star ratings and written reviews strictly gated to eligible completed bookings (with double-review prevention).
- **Complaints & Dispute Raising**: Structured grievance submission with category, description, and status tracking (`OPEN`, `RESOLVED`, `REJECTED`).
- **Profile & Saved Addresses**: Multiple address management with default tagging and saved favorite providers.

### B. Service Provider
- **Provider Dashboard**: Real-time stats on pending requests, today's schedule, total completed jobs, gross earnings, and average rating.
- **Service Catalog Management**: Add, update, activate, and deactivate specialized service offerings and custom pricing.
- **Working Hours & Availability**: Configurable weekly schedule (start/end times per day of week), visit fees, and blackout dates.
- **Booking Request Workflow**: Accept or reject pending incoming bookings, progress accepted jobs to `IN_PROGRESS`, and mark as `COMPLETED`.
- **Earnings & Receipts**: Detailed transaction breakdown distinguishing gross booking value and net provider payout.
- **Direct Customer Chat**: Integrated message thread per booking.

### C. Administrator
- **Platform Analytics**: Database-calculated metrics for gross booking volume, platform revenue, total customers, verified providers, and completion/cancellation rates.
- **Provider Verification Queue**: Review pending provider credentials, business licenses, and approve or reject onboarding applications.
- **User Directory**: Centralized directory of all registered users with role-switching and account deactivation capabilities.
- **Category & Service Oversight**: Create, edit, and organize marketplace categories.
- **Booking Oversight**: System-wide booking monitor with status inspection and intervention.
- **Dispute & Complaint Resolution**: Review customer grievances, record administrative resolution notes, and resolve cases.
- **Review Moderation**: Hide or reinstate customer reviews to maintain marketplace integrity.
- **Administrative Audit Logs**: Immutable chronological record of administrative actions, timestamps, and IP addresses.
- **Platform Policy Settings**: Configurable platform fee percentages, cancellation cutoff windows (e.g. 2 hours), and support contact channels.

---

## 2. Technology Stack

| Layer | Technologies & Tools |
| :--- | :--- |
| **Frontend SPA** | React 19, Vite 8, React Router v7, Lucide React, Custom CSS Design System |
| **Backend REST API** | Node.js (v20+ / v24+), Express.js 4.21, CORS, Cookie-Parser, Supertest |
| **Database** | SQLite 3 via `better-sqlite3` v13, WAL Mode, Foreign Key Enforcement, ACID Transactions |
| **Security & Auth** | JSON Web Tokens (`jsonwebtoken`), `bcryptjs` (salt rounds: 10), HTTP-only cookies, Rate Limiting, Role-Based Access Control (RBAC) |
| **Testing** | Node.js native test runner (`node --test`), Supertest API suite |

---

## 3. Architecture & Concurrency Protection

### Concurrency-Safe Double Booking Prevention
To prevent two customers from simultaneously reserving overlapping time slots with the same provider:
1. When a booking request is received at `POST /api/bookings`, an immediate SQLite transaction begins (`db.transaction`).
2. An SQL query with interval-overlap logic is evaluated:
   ```sql
   SELECT id FROM bookings
   WHERE provider_id = ?
     AND booking_date = ?
     AND status IN ('PENDING', 'CONFIRMED', 'IN_PROGRESS')
     AND NOT (end_time <= ? OR start_time >= ?)
   ```
3. If any conflict exists, the transaction aborts and returns an `HTTP 409 Conflict` error with a clear message: `"This time slot is no longer available. A conflicting reservation already exists."`
4. Only non-colliding reservations are persisted, with full financial snapshots stored in the same atomic transaction.

### Booking State Machine
Strict transitions are enforced server-side:
- `PENDING` ➔ `CONFIRMED`, `REJECTED`, or `CANCELLED`
- `CONFIRMED` ➔ `IN_PROGRESS` or `CANCELLED` (subject to cancellation cutoff rules)
- `IN_PROGRESS` ➔ `COMPLETED`
- `COMPLETED` ➔ Terminal state (eligible for reviews and receipts)
- `REJECTED` / `CANCELLED` ➔ Terminal states

---

## 4. Database Schema

The database consists of 22 normalized relational tables defined in `server/src/db/schema.sql`:
1. `users`: Authentication records, email, bcrypt hash, role (`CUSTOMER`, `PROVIDER`, `ADMIN`), active status.
2. `customer_profiles`: Customer personal details and phone numbers.
3. `provider_profiles`: Provider business name, bio, experience years, visit fee, verification status (`PENDING`, `APPROVED`, `REJECTED`, `SUSPENDED`).
4. `addresses`: Normalized customer service addresses.
5. `service_categories`: Marketplace categories (Plumbing, Electrical, Cleaning, etc.).
6. `services`: Specific offerings linked to categories and providers with duration and pricing.
7. `service_areas`: Zip codes/cities serviced by providers.
8. `provider_working_hours`: Weekly schedule per day (0–6) with working hours.
9. `provider_unavailable_dates`: Provider vacation and blackout dates.
10. `bookings`: Master booking record with unique booking reference, snapshot prices, date, time slots, and status.
11. `booking_status_history`: Audit trail of all state transitions with timestamps and actors.
12. `payments`: Payment records (`PENDING`, `PAID`, `FAILED`, `REFUNDED`), method (`CASH_ON_DELIVERY`, `MOCK_CARD`), transaction references.
13. `refunds`: Refund requests and approval history.
14. `reviews`: 1-to-5 star ratings and feedback (strictly one per completed booking).
15. `favorites`: Customer saved provider relations.
16. `notifications`: In-app event alerts with read/unread tracking.
17. `conversations`: Scoped direct chat threads tied to bookings.
18. `messages`: Individual chat entries with timestamps.
19. `complaints`: Dispute cases raised by customers.
20. `provider_verification_documents`: License/ID metadata submitted by providers.
21. `admin_audit_logs`: Administrative action logs.
22. `platform_settings`: Dynamic marketplace fee rates, cancellation policies, and contact information.

---

## 5. Demo Credentials

The database is pre-seeded with realistic records across all roles. The Login page also features convenient **Demo Account Quick-Fill Buttons** for rapid testing:

| Role | Email | Password | Details |
| :--- | :--- | :--- | :--- |
| **Administrator** | `admin@servicenest.com` | `AdminPass123!` | Full admin dashboard, verification queue, disputes, audit logs |
| **Service Provider** | `john.sparks@nest.com` | `ProviderPass123!` | Approved Electrician (Apex Electrical), manage bookings, schedule, earnings |
| **Customer** | `sarah.customer@nest.com` | `CustomerPass123!` | Active customer with pre-existing booking history and saved addresses |
| **Customer 2** | `michael.scott@nest.com` | `CustomerPass123!` | Secondary customer profile |
| **Pending Provider** | `alex.pest@nest.com` | `ProviderPass123!` | SafeShield Pest Control (Pending verification queue) |

---

## 6. Quick Start & Installation

### Prerequisites
- **Node.js**: v20.x, v22.x, or v24.x
- **npm**: v10.x or higher

### Step 1: Clone and Install Dependencies
From the repository root:
```bash
# Install root, backend, and frontend dependencies in one command
npm run install:all
```

### Step 2: Configure Environment Variables
Create a `.env` file in the root directory (or use default development fallbacks):
```bash
cp .env.example .env
```
Default configuration:
```env
PORT=5000
NODE_ENV=development
JWT_SECRET=super_secret_jwt_servicenest_key_change_in_production
CLIENT_URL=http://localhost:5173
DATABASE_PATH=./src/db/servicenest.db
CANCELLATION_WINDOW_HOURS=2
PLATFORM_FEE_PERCENTAGE=5
```

### Step 3: Initialize and Seed Database
```bash
# Initialize schema and load 10 categories, 11 providers, bookings, reviews & messages
npm run seed
```

### Step 4: Run Application in Development Mode
```bash
npm run dev
```
This runs both processes concurrently:
- **Backend API**: `http://localhost:5000` (Health check: `http://localhost:5000/api/health`)
- **Frontend SPA**: `http://localhost:5173`

---

## 7. Running Tests

ServiceNest includes automated unit, integration, RBAC, and end-to-end journey test suites:

```bash
npm test
```

### Test Coverage Highlights:
- **`auth.test.js`**: Customer registration, duplicate email rejection (409), password length validation (400), login authentication, and `/api/auth/me` token inspection.
- **`booking-concurrency.test.js`**: Slot interval mathematical overlap checks, atomic double-booking rejection with HTTP 409 Conflict, and invalid state transition guards.
- **`rbac-reviews.test.js`**: RBAC route protection (customers blocked from admin endpoints with HTTP 403), premature review rejection for uncompleted bookings (HTTP 400).
- **`e2e-journeys.test.js`**: Full multi-role lifecycle — customer creates booking, concurrent collision rejected, in-app messaging exchanged, provider accepts & completes job, customer reviews, duplicate review rejected, admin checks live financial statistics and audit logs.

---

## 8. API Documentation Overview

All API endpoints follow RESTful standards with consistent JSON responses:
- Success: `{ "success": true, "data": ... }`
- Error: `{ "success": false, "message": "...", "errors": null }`

For complete parameter specifications, query filters, and sample request/response payloads, consult:
📄 **[`API_DOCUMENTATION.md`](./API_DOCUMENTATION.md)**

---

## 9. Git & GitHub Setup

To push this codebase to your GitHub account:

```bash
# 1. Initialize git repository
git init

# 2. Stage all files (respecting .gitignore)
git add .

# 3. Create initial commit
git commit -m "feat: initial commit of ServiceNest full-stack booking application"

# 4. Link your GitHub repository
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/servicenest.git

# 5. Push to GitHub
git push -u origin main
```

---

## 10. Production Deployment Guide

### Architecture for Production
In a standard production environment:
1. **Frontend**: Build the static bundle (`npm --prefix client run build`) and serve via Nginx, Cloudflare Pages, or AWS S3/CloudFront.
2. **Backend**: Run Node.js with a process manager like **PM2** behind an Nginx reverse proxy with SSL/TLS (Let's Encrypt).
3. **Database**: SQLite WAL mode with periodic snapshots, or migrate to PostgreSQL if multi-instance clustering is required.

### Step-by-Step Production Setup on Ubuntu/Debian:

```bash
# 1. Build frontend bundle
cd client && npm run build && cd ..

# 2. Setup environment variables
cat << 'EOF' > server/.env
NODE_ENV=production
PORT=5000
JWT_SECRET=generate_a_random_64_char_secret_key_here
CLIENT_URL=https://servicenest.yourdomain.com
DATABASE_PATH=./src/db/servicenest.db
CANCELLATION_WINDOW_HOURS=2
PLATFORM_FEE_PERCENTAGE=5
EOF

# 3. Initialize production database
npm --prefix server run init-db
npm --prefix server run seed

# 4. Start backend using PM2
npm install -g pm2
pm2 start server/src/server.js --name "servicenest-api"
pm2 save
pm2 startup
```

### Sample Nginx Configuration (`/etc/nginx/sites-available/servicenest`):
```nginx
server {
    listen 80;
    server_name servicenest.yourdomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name servicenest.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/servicenest.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/servicenest.yourdomain.com/privkey.pem;

    # Frontend Static Files
    location / {
        root /var/www/servicenest/client/dist;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    # Backend API Proxy
    location /api/ {
        proxy_pass http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

## 11. Production-Readiness Checklist

- [x] **Secure Password Storage**: Passwords hashed with `bcryptjs` (salt rounds: 10). Plaintext passwords never stored or logged.
- [x] **Role-Based Authorization**: RBAC middleware strictly enforces permissions on every protected endpoint.
- [x] **Parameterized Queries**: All database queries use SQLite parameterized statements (`?`), eliminating SQL injection.
- [x] **Concurrency & Race Conditions**: Immediate database transactions with interval overlap algorithms prevent double-booking.
- [x] **Price Immutability**: Service price snapshots preserved in bookings against future price edits.
- [x] **Input Validation & Sanitization**: Server-side request body and query parameter validation on all routes.
- [x] **Rate Limiting**: Sliding window rate limiting on authentication routes prevents brute-force attacks.
- [x] **CORS & Cookies**: Credentials-enabled CORS restricted to client origin; JWT token transport support via secure HTTP headers.
- [x] **Zero External Key Dependencies**: Runs self-contained without requiring paid third-party APIs or AI keys.
- [x] **Responsive Mobile Design**: Fluid layouts tested across desktop, tablet, and mobile breakpoints without horizontal scroll.
- [x] **Data Persistence**: SQLite database with WAL mode ensures durability across restarts and process reloads.
- [x] **Automated Test Coverage**: 100% pass rate across authentication, RBAC, booking collisions, and complete user journeys.

---

## License
MIT License. Built for ServiceNest Local Service Marketplace.
