# Backend Feature Requirements Specifications

## 1. User Authentication & Management System

### Overview
Secure authentication system supporting email/password and OAuth (Google, Facebook) with JWT-based session management.

### API Endpoints

#### 1.1 User Registration
```
POST /api/v1/auth/register
```

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "firstName": "John",
  "lastName": "Doe",
  "userType": "guest|host",
  "phone": "+1234567890"
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "userId": "uuid",
    "email": "user@example.com",
    "accessToken": "jwt_token",
    "refreshToken": "refresh_token",
    "emailVerified": false
  }
}
```

**Validation Rules:**
- Email: Valid format, unique, max 255 chars
- Password: Min 8 chars, 1 uppercase, 1 lowercase, 1 number, 1 special char
- Phone: Valid international format (optional)
- UserType: Must be "guest" or "host"

**Error Responses:**
- 400: Invalid input data
- 409: Email already exists
- 500: Server error

---

#### 1.2 User Login
```
POST /api/v1/auth/login
```

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123!"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "userId": "uuid",
    "accessToken": "jwt_token",
    "refreshToken": "refresh_token",
    "user": {
      "email": "user@example.com",
      "firstName": "John",
      "userType": "guest"
    }
  }
}
```

**Validation Rules:**
- Rate limiting: 5 attempts per 15 minutes per IP
- Account lockout after 5 failed attempts (30 minutes)
- Email & password required

---

#### 1.3 OAuth Login
```
POST /api/v1/auth/oauth/{provider}
GET /api/v1/auth/oauth/{provider}/callback
```

**Supported Providers:** `google`, `facebook`

**Request Body (POST):**
```json
{
  "code": "oauth_authorization_code",
  "redirectUri": "https://app.com/callback"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "userId": "uuid",
    "accessToken": "jwt_token",
    "refreshToken": "refresh_token",
    "isNewUser": true
  }
}
```

---

#### 1.4 Token Refresh
```
POST /api/v1/auth/refresh
```

**Request Body:**
```json
{
  "refreshToken": "refresh_token_string"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "accessToken": "new_jwt_token",
    "refreshToken": "new_refresh_token"
  }
}
```

---

#### 1.5 Email Verification
```
POST /api/v1/auth/verify-email
```

**Request Body:**
```json
{
  "token": "verification_token"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Email verified successfully"
}
```

---

### Performance Criteria
- **Response Time:** < 200ms for login/register (95th percentile)
- **Token Expiry:** Access token: 15 minutes, Refresh token: 30 days
- **Concurrent Users:** Support 10,000 simultaneous authentications
- **Password Hashing:** bcrypt with salt rounds = 12
- **Database Queries:** Indexed on email (lookup < 10ms)

---

### Security Requirements
- **Password Storage:** bcrypt hashing, never store plain text
- **JWT Signing:** RS256 algorithm with rotating keys
- **HTTPS Only:** All endpoints require TLS 1.3
- **CORS:** Whitelist specific origins only
- **Rate Limiting:** 100 requests/minute per user
- **Session Management:** Automatic logout after 24 hours inactivity

---

## 2. Property Management System

### Overview
Complete CRUD operations for property listings with image upload, availability calendar, and search optimization.

### API Endpoints

#### 2.1 Create Listing
```
POST /api/v1/properties
Authorization: Bearer {token}
```

**Request Body (multipart/form-data):**
```json
{
  "title": "Cozy Beach House",
  "description": "Beautiful 3BR house...",
  "location": {
    "address": "123 Ocean Drive",
    "city": "Miami",
    "state": "FL",
    "country": "USA",
    "zipCode": "33139",
    "latitude": 25.7617,
    "longitude": -80.1918
  },
  "pricePerNight": 250.00,
  "currency": "USD",
  "maxGuests": 6,
  "bedrooms": 3,
  "bathrooms": 2,
  "amenities": ["wifi", "pool", "parking", "kitchen"],
  "propertyType": "house",
  "images": ["file1.jpg", "file2.jpg"]
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "propertyId": "uuid",
    "title": "Cozy Beach House",
    "status": "active",
    "createdAt": "2025-11-02T10:00:00Z",
    "imageUrls": ["https://cdn.../img1.jpg"]
  }
}
```

**Validation Rules:**
- Title: 10-100 chars, required
- Description: 50-2000 chars, required
- Price: Positive decimal, max 2 decimals, required
- MaxGuests: Integer 1-50, required
- Images: Max 10 images, each < 5MB, formats: JPG/PNG/WebP
- Amenities: From predefined list only
- Location: Valid coordinates, required fields

---

#### 2.2 Update Listing
```
PUT /api/v1/properties/{propertyId}
Authorization: Bearer {token}
```

**Request Body:** (Same as create, all fields optional)

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "propertyId": "uuid",
    "updatedAt": "2025-11-02T11:00:00Z"
  }
}
```

**Authorization:** Only property owner can update

---

#### 2.3 Delete Listing
```
DELETE /api/v1/properties/{propertyId}
Authorization: Bearer {token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Property deleted successfully"
}
```

**Business Rules:**
- Cannot delete if active bookings exist
- Soft delete (mark as inactive)
- Only owner or admin can delete

---

#### 2.4 Get Property Details
```
GET /api/v1/properties/{propertyId}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "propertyId": "uuid",
    "title": "Cozy Beach House",
    "description": "...",
    "location": {...},
    "pricePerNight": 250.00,
    "maxGuests": 6,
    "amenities": [],
    "images": [],
    "averageRating": 4.8,
    "totalReviews": 42,
    "host": {
      "userId": "uuid",
      "firstName": "Jane",
      "profileImage": "url"
    },
    "availability": {
      "blockedDates": ["2025-12-25"]
    }
  }
}
```

---

#### 2.5 Search Properties
```
GET /api/v1/properties/search
```

**Query Parameters:**
```
location=Miami&
checkIn=2025-12-20&
checkOut=2025-12-25&
guests=4&
minPrice=100&
maxPrice=500&
amenities=wifi,pool&
propertyType=house&
page=1&
limit=20&
sortBy=price&
sortOrder=asc
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "properties": [...],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 150,
      "totalPages": 8
    },
    "filters": {
      "appliedFilters": {...}
    }
  }
}
```

**Validation Rules:**
- checkIn < checkOut, both required for availability filter
- guests: Integer 1-50
- price: Positive decimals
- page: Min 1, limit: Max 100
- Dates: ISO 8601 format

---

#### 2.6 Manage Availability
```
PUT /api/v1/properties/{propertyId}/availability
Authorization: Bearer {token}
```

**Request Body:**
```json
{
  "blockedDates": [
    {
      "startDate": "2025-12-24",
      "endDate": "2025-12-26",
      "reason": "personal use"
    }
  ]
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Availability updated"
}
```

---

### Performance Criteria
- **Search Response:** < 500ms for filtered search (95th percentile)
- **Image Upload:** < 5 seconds per image with CDN distribution
- **Database Indexing:** location (geospatial), price, amenities
- **Caching:** Property details cached for 5 minutes
- **Pagination:** Max 100 results per page
- **Concurrent Writes:** Support 1,000 simultaneous property updates

---

### Data Integrity
- **Image Storage:** AWS S3 / Cloudinary with CDN
- **Geocoding:** Validate addresses with Google Maps API
- **Soft Deletes:** Maintain historical data for analytics
- **Audit Trail:** Log all CRUD operations with timestamps

---

## 3. Booking System

### Overview
End-to-end booking management with availability validation, payment integration, and status tracking.

### API Endpoints

#### 3.1 Create Booking
```
POST /api/v1/bookings
Authorization: Bearer {token}
```

**Request Body:**
```json
{
  "propertyId": "uuid",
  "checkInDate": "2025-12-20",
  "checkOutDate": "2025-12-25",
  "numberOfGuests": 4,
  "specialRequests": "Early check-in",
  "paymentMethodId": "pm_stripe_xxx"
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "bookingId": "uuid",
    "status": "pending_payment",
    "totalAmount": 1375.00,
    "breakdown": {
      "subtotal": 1250.00,
      "serviceFee": 87.50,
      "taxes": 37.50
    },
    "paymentIntentId": "pi_stripe_xxx",
    "expiresAt": "2025-11-02T10:15:00Z"
  }
}
```

**Validation Rules:**
- Property must exist and be active
- Dates must not conflict with existing bookings
- checkInDate > current date
- checkOutDate > checkInDate
- numberOfGuests <= property.maxGuests
- User must be authenticated guest

**Business Rules:**
- Lock dates for 15 minutes during payment
- Auto-cancel if payment not completed in 15 minutes
- Minimum 1-night stay

---

#### 3.2 Confirm Payment
```
POST /api/v1/bookings/{bookingId}/confirm-payment
Authorization: Bearer {token}
```

**Request Body:**
```json
{
  "paymentIntentId": "pi_stripe_xxx",
  "paymentStatus": "succeeded"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "bookingId": "uuid",
    "status": "confirmed",
    "confirmationCode": "ABC123XYZ",
    "receiptUrl": "https://..."
  }
}
```

**Actions Triggered:**
- Update booking status to "confirmed"
- Mark dates as unavailable
- Send confirmation emails to guest & host
- Schedule payout to host
- Generate booking receipt

---

#### 3.3 Cancel Booking
```
POST /api/v1/bookings/{bookingId}/cancel
Authorization: Bearer {token}
```

**Request Body:**
```json
{
  "reason": "Change of plans",
  "cancelledBy": "guest"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "bookingId": "uuid",
    "status": "cancelled",
    "refundAmount": 937.50,
    "refundPolicy": "flexible",
    "refundProcessedAt": "2025-11-02T10:30:00Z"
  }
}
```

**Cancellation Policies:**
- **Flexible:** Full refund if cancelled 24+ hours before check-in
- **Moderate:** 50% refund if cancelled 7+ days before check-in
- **Strict:** No refund within 14 days of check-in

**Authorization:**
- Guest can cancel any time
- Host can cancel with valid reason (emergency)

---

#### 3.4 Get Booking Details
```
GET /api/v1/bookings/{bookingId}
Authorization: Bearer {token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "bookingId": "uuid",
    "property": {...},
    "guest": {...},
    "checkInDate": "2025-12-20",
    "checkOutDate": "2025-12-25",
    "numberOfGuests": 4,
    "status": "confirmed",
    "totalAmount": 1375.00,
    "paymentStatus": "paid",
    "specialRequests": "...",
    "createdAt": "...",
    "cancellationPolicy": "flexible"
  }
}
```

---

#### 3.5 List User Bookings
```
GET /api/v1/bookings
Authorization: Bearer {token}
```

**Query Parameters:**
```
status=confirmed&
type=upcoming&
page=1&
limit=20
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "bookings": [...],
    "pagination": {...}
  }
}
```

**Filter Options:**
- **status:** pending, confirmed, cancelled, completed
- **type:** upcoming, past, all
- **dateRange:** Custom date filtering

---

#### 3.6 Get Property Bookings (Host)
```
GET /api/v1/properties/{propertyId}/bookings
Authorization: Bearer {token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "bookings": [...],
    "occupancyRate": 75.5,
    "upcomingBookings": 5
  }
}
```

**Authorization:** Only property owner

---

### Performance Criteria
- **Availability Check:** < 100ms (critical for UX)
- **Booking Creation:** < 2 seconds end-to-end
- **Payment Processing:** < 5 seconds with Stripe webhook
- **Concurrent Bookings:** Handle 5,000 simultaneous requests
- **Date Locking:** Redis-based with 15-minute TTL
- **Database Transactions:** ACID compliance for booking + payment

---

### Concurrency Control
- **Pessimistic Locking:** Lock property dates during booking creation
- **Optimistic Locking:** Version control on booking records
- **Idempotency:** All POST requests use idempotency keys
- **Race Condition Prevention:** Database-level unique constraints on date ranges

---

### Integration Requirements
- **Payment Gateway:** Stripe/PayPal webhook integration
- **Email Service:** SendGrid for transactional emails
- **Calendar Sync:** iCal export for external calendars
- **Notification Queue:** RabbitMQ for async notifications

---


