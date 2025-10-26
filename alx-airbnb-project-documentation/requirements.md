# AirBnB Backend - Feature Requirements Specification

**Version:** 1.0 | **Date:** October 26, 2024

---

## 1. User Authentication System

### 1.1 Overview
JWT-based stateless authentication with bcrypt password hashing (12 rounds). Supports registration, login, token refresh, and password reset.

### 1.2 API Endpoints

#### **POST /api/v1/auth/register**
Creates new user account.

**Request:**
```json
{
  "first_name": "string (2-50 chars)",
  "last_name": "string (2-50 chars)", 
  "email": "string (unique, valid format)",
  "password": "string (min 8, 1 upper, 1 lower, 1 digit, 1 special)",
  "phone_number": "string (E.164 format, optional)",
  "role": "enum (guest|host, default: guest)"
}
```

**Response (201):**
```json
{
  "status": "success",
  "data": {
    "user": {
      "user_id": "uuid",
      "first_name": "string",
      "last_name": "string",
      "email": "string",
      "role": "string",
      "email_verified": false,
      "created_at": "timestamp"
    }
  }
}
```

**Validation:**
- Email: RFC 5322 compliant, unique, valid MX records
- Password: 8-128 chars, complexity requirements, not in common password list
- Names: No numbers/special chars except spaces, hyphens, apostrophes

---

#### **POST /api/v1/auth/login**
Authenticates user and returns JWT tokens.

**Request:**
```json
{
  "email": "string",
  "password": "string",
  "remember_me": "boolean (optional)"
}
```

**Response (200):**
```json
{
  "status": "success",
  "data": {
    "user": { "user_id": "uuid", "first_name": "string", "email": "string", "role": "string" },
    "tokens": {
      "access_token": "jwt_string",
      "refresh_token": "jwt_string",
      "token_type": "Bearer",
      "expires_in": 3600
    }
  }
}
```

**Security:**
- Rate limit: 5 attempts per 15 min per IP
- Account lock: 15 min after 5 failed attempts
- Tokens: Access (1hr), Refresh (7 days, 30 days if remember_me)

---

#### **POST /api/v1/auth/refresh**
Generates new access token.

**Request:** `{ "refresh_token": "string" }`  
**Response (200):** `{ "access_token": "jwt", "expires_in": 3600 }`

#### **POST /api/v1/auth/logout**
Invalidates tokens. Requires `Authorization: Bearer {token}`

#### **GET /api/v1/auth/me**
Returns current user profile. Requires authentication.

#### **PATCH /api/v1/auth/me**
Updates user profile (first_name, last_name, phone_number).

#### **PUT /api/v1/auth/password**
Changes password. Requires current_password, new_password, confirm_password.

#### **POST /api/v1/auth/password-reset/request**
Sends reset email. Request: `{ "email": "string" }`

#### **POST /api/v1/auth/password-reset/confirm**
Resets password using token.

### 1.3 Performance Criteria
- Login: < 300ms (target), < 600ms (max)
- Register: < 500ms (target), < 1000ms (max)
- Token refresh: < 100ms (target), < 200ms (max)
- Throughput: 1000 login req/s, 2000 refresh req/s

### 1.4 Error Codes
| Code | Status | Description |
|------|--------|-------------|
| VALIDATION_ERROR | 400 | Invalid input |
| EMAIL_EXISTS | 409 | Duplicate email |
| INVALID_CREDENTIALS | 401 | Wrong email/password |
| RATE_LIMIT_EXCEEDED | 429 | Too many attempts |
| TOKEN_EXPIRED | 401 | JWT expired |
| EMAIL_NOT_VERIFIED | 403 | Verification required |

---

## 2. Property Management System

### 2.1 Overview
Manages property listings with location-based search, amenities, pricing, and media. Uses Elasticsearch for search, Redis for caching, S3 for media storage.

### 2.2 API Endpoints

#### **POST /api/v1/properties**
Creates property listing. Requires host/admin role.

**Request:**
```json
{
  "name": "string (10-100 chars)",
  "description": "string (50-5000 chars)",
  "property_type": "enum (apartment|house|condo|villa|cabin)",
  "location": {
    "street_address": "string",
    "city": "string",
    "state_province": "string",
    "country": "string",
    "postal_code": "string",
    "latitude": "decimal (optional)",
    "longitude": "decimal (optional)"
  },
  "pricepernight": "decimal (10-100000, 2 decimals)",
  "bedrooms": "integer (0-50)",
  "bathrooms": "integer (0-50)",
  "max_guests": "integer (1-100)",
  "amenities": ["uuid array"],
  "cancellation_policy": "enum (flexible|moderate|strict)",
  "check_in_time": "string (HH:MM)",
  "check_out_time": "string (HH:MM)"
}
```

**Response (201):** Property object with assigned property_id, location_id, status=active

**Validation:**
- pricepernight: >= $10, <= $100,000
- max_guests: >= (bedrooms × 2)
- check_out_time > check_in_time
- amenities: Valid UUID references

---

#### **GET /api/v1/properties**
Search/filter properties with pagination.

**Query Parameters:**
- Pagination: `page` (default: 1), `limit` (default: 20, max: 100)
- Filters: `city`, `country`, `min_price`, `max_price`, `bedrooms`, `bathrooms`, `guests`, `property_type`, `amenities` (comma-separated)
- Geo: `lat`, `lng`, `radius` (km)
- Availability: `start_date`, `end_date` (YYYY-MM-DD)
- Sort: `sort_by` (price|rating|created_at), `sort_order` (asc|desc)
- Search: `search` (full-text in name/description)

**Response (200):**
```json
{
  "status": "success",
  "data": {
    "properties": [{
      "property_id": "uuid",
      "name": "string",
      "property_type": "string",
      "location": { "city": "string", "country": "string", "latitude": 0, "longitude": 0 },
      "pricepernight": 0,
      "bedrooms": 0,
      "bathrooms": 0,
      "max_guests": 0,
      "images": [{ "url": "string", "thumbnail_url": "string", "is_primary": true }],
      "average_rating": 0,
      "total_reviews": 0,
      "host": { "host_id": "uuid", "first_name": "string", "superhost": false }
    }],
    "pagination": {
      "current_page": 1,
      "total_pages": 10,
      "total_items": 200,
      "has_next": true,
      "has_prev": false
    }
  }
}
```

---

#### **GET /api/v1/properties/:property_id**
Get single property details.

**Response (200):** Full property object including amenities, house_rules, host info, reviews summary

#### **PATCH /api/v1/properties/:property_id**
Update property. Requires host ownership or admin role.

**Request:** Any fields from creation (partial update allowed)

#### **DELETE /api/v1/properties/:property_id**
Soft delete (set status=archived). Requires host ownership or admin.

#### **POST /api/v1/properties/:property_id/images**
Upload property images (multipart/form-data). Max 20 images per property, 5MB each.

**Request:** `Content-Type: multipart/form-data`, Field: `images[]`

**Response (201):** Array of uploaded image objects with CDN URLs

### 2.3 Performance Criteria
- Create property: < 800ms
- Search properties: < 500ms (cached), < 2000ms (uncached with complex filters)
- Get single property: < 200ms (cached), < 600ms (uncached)
- Image upload: < 3000ms per image (with processing)
- Throughput: 500 search req/s, 100 create req/s

---

## 3. Booking System

### 3.1 Overview
Manages property reservations with availability checking, pricing calculation, payment integration, and booking lifecycle.

### 3.2 API Endpoints

#### **POST /api/v1/bookings**
Create new booking reservation.

**Request:**
```json
{
  "property_id": "uuid",
  "start_date": "date (YYYY-MM-DD)",
  "end_date": "date (YYYY-MM-DD)"
}
```

**Validation:**
- Dates: end_date > start_date, start_date >= today, duration <= 365 days
- Availability: No overlapping confirmed bookings for property
- User: Cannot book own property

**Response (201):**
```json
{
  "status": "success",
  "data": {
    "booking": {
      "booking_id": "uuid",
      "property_id": "uuid",
      "user_id": "uuid",
      "start_date": "date",
      "end_date": "date",
      "nights": 7,
      "total_price": 2850.00,
      "price_breakdown": [
        { "item_type": "base_rate", "description": "7 nights × $350", "amount": 2450 },
        { "item_type": "cleaning_fee", "amount": 150 },
        { "item_type": "service_fee", "amount": 260 },
        { "item_type": "tax", "amount": 196 },
        { "item_type": "discount", "amount": -206 }
      ],
      "status": "pending",
      "created_at": "timestamp"
    }
  }
}
```

**Business Logic:**
- Calculate: base_rate + cleaning_fee + service_fee (10%) + tax (8%)
- Apply discounts: 7+ nights (5%), 30+ nights (15%)
- Status: pending → confirmed (after payment) | canceled

---

#### **GET /api/v1/bookings**
List user's bookings (guest or host view).

**Query Parameters:** `status` (pending|confirmed|canceled), `role` (guest|host), `page`, `limit`

**Response (200):** Paginated booking list with property and user details

#### **GET /api/v1/bookings/:booking_id**
Get booking details. Requires booking ownership or host/admin role.

#### **PATCH /api/v1/bookings/:booking_id**
Update booking status. Only status=canceled allowed by users.

**Request:** `{ "status": "canceled", "cancellation_reason": "string (optional)" }`

**Response (200):** Updated booking with canceled_at timestamp

#### **POST /api/v1/bookings/:booking_id/payment**
Process booking payment.

**Request:**
```json
{
  "payment_method": "enum (credit_card|paypal|stripe)",
  "payment_details": {
    "token": "string (payment gateway token)"
  }
}
```

**Response (200):**
```json
{
  "status": "success",
  "data": {
    "payment": {
      "payment_id": "uuid",
      "booking_id": "uuid",
      "amount": 2850.00,
      "payment_method": "credit_card",
      "transaction_id": "ext_12345",
      "status": "completed",
      "payment_date": "timestamp"
    },
    "booking": {
      "booking_id": "uuid",
      "status": "confirmed"
    }
  }
}
```

**Payment Flow:**
1. Validate booking exists and status=pending
2. Call payment gateway API
3. On success: Insert Payment record, Update Booking status=confirmed, Send confirmation emails
4. On failure: Return error, keep booking status=pending

#### **GET /api/v1/properties/:property_id/availability**
Check property availability for date range.

**Query:** `start_date`, `end_date`

**Response (200):**
```json
{
  "available": true,
  "blocked_dates": ["2024-11-01", "2024-11-02"],
  "price_per_night": 350.00
}
```

### 3.3 Performance Criteria
- Create booking: < 1000ms (includes availability check)
- Payment processing: < 5000ms (external API dependency)
- Availability check: < 300ms
- List bookings: < 500ms
- Throughput: 200 booking req/s, 50 payment req/s

### 3.4 Constraints
- Database trigger: Prevent overlapping bookings (property_id + date range)
- Race condition handling: Row-level locking on availability check
- Payment timeout: 15 minutes to complete payment after booking creation
- Auto-cancel: Pending bookings > 15 min old without payment

---

## Common Specifications

### Error Response Format
```json
{
  "status": "error",
  "message": "Human-readable message",
  "code": "ERROR_CODE",
  "errors": [{ "field": "field_name", "message": "Error detail" }],
  "timestamp": "ISO 8601",
  "path": "/api/v1/endpoint"
}
```

### HTTP Status Codes
- 200: Success
- 201: Created
- 400: Bad Request (validation)
- 401: Unauthorized
- 403: Forbidden
- 404: Not Found
- 409: Conflict (duplicate)
- 429: Rate Limited
- 500: Server Error

### Rate Limiting (Global)
- Authenticated: 1000 req/min per user
- Unauthenticated: 100 req/min per IP
- Payment endpoints: 10 req/min per user

### Caching Strategy
- Properties list: 5 min TTL
- Single property: 15 min TTL
- User profile: 10 min TTL
- Availability: 1 min TTL
- Cache invalidation: On update/delete operations
