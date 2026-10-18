# ServiceNest REST API Documentation

This document describes the REST API endpoints provided by the **ServiceNest** server.

Base URL: `http://localhost:5000/api`

---

## 1. Authentication (`/api/auth`)

### Register
- **URL**: `POST /api/auth/register`
- **Rate Limit**: 30 requests / 15 minutes
- **Request Body (Customer)**:
  ```json
  {
    "name": "Sarah Jenkins",
    "email": "sarah.customer@nest.com",
    "password": "CustomerPass123!",
    "role": "CUSTOMER",
    "phone": "+1 (555) 234-5678",
    "street": "742 Evergreen Terrace",
    "unit": "Apt 4B",
    "city": "New York",
    "postalCode": "10001"
  }
  ```
- **Request Body (Provider)**:
  ```json
  {
    "name": "John Sparks",
    "email": "john.sparks@nest.com",
    "password": "ProviderPass123!",
    "role": "PROVIDER",
    "businessName": "Apex Electrical Solutions",
    "bio": "Certified Master Electrician with 12+ years experience.",
    "experienceYears": 12,
    "licenseNumber": "ELEC-NY-99214",
    "city": "New York"
  }
  ```
- **Response**: `201 Created`
  ```json
  {
    "success": true,
    "message": "Customer registration successful!",
    "token": "eyJhbGciOiJIUzI1NiIsIn...",
    "user": {
      "id": 2,
      "name": "Sarah Jenkins",
      "email": "sarah.customer@nest.com",
      "role": "CUSTOMER",
      "phone": "+1 (555) 234-5678",
      "status": "ACTIVE"
    }
  }
  ```

### Login
- **URL**: `POST /api/auth/login`
- **Request Body**:
  ```json
  {
    "email": "sarah.customer@nest.com",
    "password": "CustomerPass123!"
  }
  ```
- **Response**: `200 OK`
  ```json
  {
    "success": true,
    "message": "Login successful.",
    "token": "eyJhbGciOiJIUzI1NiIsIn...",
    "user": { ... }
  }
  ```

### Current User
- **URL**: `GET /api/auth/me`
- **Headers**: `Authorization: Bearer <token>`
- **Response**: `200 OK`

### Logout
- **URL**: `POST /api/auth/logout`
- **Response**: `200 OK` (clears cookie)

---

## 2. Categories & Services (`/api/categories`, `/api/services`)

### List Categories
- **URL**: `GET /api/categories`
- **Response**: List of active categories with count of active services.

### Discover Services (Search & Filter)
- **URL**: `GET /api/services`
- **Query Parameters**:
  - `q`: Search keyword in title, description, or provider name
  - `category`: Category slug (e.g., `electrical`, `plumbing`)
  - `city`: City name (e.g., `New York`, `Brooklyn`)
  - `minPrice`: Minimum base rate
  - `maxPrice`: Maximum base rate
  - `minRating`: Minimum average provider rating (e.g. `4.5`)
  - `verifiedOnly`: `true` for ID & license verified providers only
  - `date`: `YYYY-MM-DD` (filters out providers unavailable on date)
  - `sortBy`: `rating_desc`, `price_asc`, `price_desc`, `popular_desc`, `newest`
  - `page`: Page index (default: `1`)
  - `limit`: Items per page (default: `12`)
- **Response**: `200 OK` with paginated services.

### Service Details
- **URL**: `GET /api/services/:id`
- **Response**: Full details, duration, visit fee, provider profile, and verified customer reviews.

---

## 3. Provider Availability (`/api/providers/:id/availability`)

- **URL**: `GET /api/providers/:id/availability?date=YYYY-MM-DD&serviceId=123`
- **Logic**:
  1. Checks blackout dates in `provider_unavailable_dates`.
  2. Retrieves working hours for date's day of week.
  3. Computes time slots based on service `duration_minutes` + `buffer_minutes`.
  4. Runs interval overlap checks against existing active bookings (`PENDING`, `CONFIRMED`, `IN_PROGRESS`).
- **Response**:
  ```json
  {
    "success": true,
    "providerId": 3,
    "date": "2026-11-25",
    "available": true,
    "workingHours": { "startTime": "08:00", "endTime": "18:00" },
    "slots": [
      { "startTime": "08:00", "endTime": "09:00", "isAvailable": true },
      { "startTime": "09:15", "endTime": "10:15", "isAvailable": false }
    ]
  }
  ```

---

## 4. Bookings & Lifecycle (`/api/bookings`)

### Create Booking
- **URL**: `POST /api/bookings`
- **Headers**: `Authorization: Bearer <customer_token>`
- **Request Body**:
  ```json
  {
    "serviceId": 1,
    "addressId": 1,
    "bookingDate": "2026-11-25",
    "startTime": "10:00",
    "customerNotes": "Main breaker keeps tripping under load"
  }
  ```
- **Concurrency Protection**:
  Executes double-booking overlap validation inside an atomic SQLite transaction lock (`NOT (end_time <= ? OR start_time >= ?)`). If any collision exists, throws `409 Conflict`.
- **Response**: `201 Created` with generated reference code e.g. `BK-202611-4821`.

### Transition Status
- **URL**: `POST /api/bookings/:id/status`
- **Allowed Transitions**:
  - `PENDING` -> `CONFIRMED`, `REJECTED`, `CANCELLED`
  - `CONFIRMED` -> `IN_PROGRESS`, `CANCELLED`
  - `IN_PROGRESS` -> `COMPLETED`
- **Cancellation Window**: Free cancellation allowed up to 2 hours prior to start time.

---

## 5. Reviews & Complaints (`/api/reviews`, `/api/complaints`)

### Submit Review
- **URL**: `POST /api/reviews`
- **Rules**:
  - Only allowed for bookings with status `COMPLETED`.
  - Max 1 review per booking.
  - Automatically recalculates provider's `average_rating` and `review_count`.

### File Complaint / Dispute
- **URL**: `POST /api/complaints`
- **Categories**: `POOR_QUALITY`, `LATE_NO_SHOW`, `UNPROFESSIONAL`, `BILLING_DISPUTE`, `PROPERTY_DAMAGE`, `OTHER`.
- **Actions**: Notifies platform administrators for dispute resolution.

---

## 6. Internal Messaging (`/api/messages`)

- `GET /api/messages/:bookingId`: Read conversation thread between customer and provider.
- `POST /api/messages/:bookingId`: Post message to provider/customer and generate in-app notification.

---

## 7. Administration (`/api/admin`)

- `GET /api/admin/overview`: Macro marketplace metrics (Gross Booking Value, Platform Fee Revenue, Completion Rate).
- `GET /api/admin/users`: User directory and status toggles (`ACTIVE` / `SUSPENDED`).
- `GET /api/admin/providers/verification-queue`: Provider license and credential inspection.
- `POST /api/admin/providers/:id/verify`: `APPROVED`, `REJECTED`, or `SUSPENDED` decisions.
- `PUT /api/admin/complaints/:id/resolve`: Mediate disputes and append resolution notes.
- `PUT /api/admin/settings`: Configure cancellation hours, platform fee percent, and auto-verification.
- `GET /api/admin/audit-logs`: Chronological log of administrative actions.
