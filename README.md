# Airbnb-Like Property Booking Backend

A production-grade Spring Boot 3 backend for a property rental platform.
Hosts list properties; guests search, book, and review them.

---

## Tech Stack

| Layer         | Technology                          |
|---------------|-------------------------------------|
| Framework     | Spring Boot 3.2.5 / Java 17         |
| Security      | Spring Security 6 + JWT (jjwt 0.11.5) |
| Persistence   | Spring Data JPA / Hibernate         |
| Database      | H2 In-Memory (zero setup needed)    |
| Docs          | Springdoc OpenAPI 2.3 / Swagger UI  |
| Build         | Maven 3.x                           |

> **MySQL / PostgreSQL** – instructions at the bottom of this file.

---

## Project Structure

```
airbnb-booking/
├── src/main/java/com/airbnb/booking/
│   ├── BookingApplication.java          ← Entry point
│   ├── config/
│   │   ├── DataInitializer.java         ← Seeds demo data on startup
│   │   ├── SecurityConfig.java
│   │   ├── OpenApiConfig.java
│   │   ├── JwtAuthenticationFilter.java
│   │   └── CustomUserDetailsService.java
│   ├── controller/
│   │   ├── AuthController.java
│   │   ├── PropertyController.java
│   │   ├── BookingController.java
│   │   └── ReviewController.java
│   ├── dto/
│   │   ├── request/   (RegisterRequest, LoginRequest, PropertyRequest,
│   │   │               AvailabilityRequest, BookingRequest, ReviewRequest)
│   │   └── response/  (ApiResponse, AuthResponse, UserResponse,
│   │                   PropertyResponse, BookingResponse,
│   │                   ReviewResponse, BookingStatisticsResponse)
│   ├── entity/        (User, Property, PropertyAvailability, Booking, Review)
│   ├── enums/         (UserRole, BookingStatus)
│   ├── exception/     (GlobalExceptionHandler + custom exceptions)
│   ├── repository/    (5 JPA repositories)
│   ├── service/       (interfaces)
│   │   └── impl/      (AuthServiceImpl, PropertyServiceImpl,
│   │                   BookingServiceImpl, ReviewServiceImpl)
│   └── util/
│       └── JwtUtil.java
└── src/main/resources/
    └── application.properties
```

---

## How to Run in Eclipse

### Prerequisites
- Java 17 (JDK)
- Maven 3.6+ (or use the Maven wrapper if added)
- Eclipse IDE for Enterprise Java (2022-09 or later)
  - with **Spring Tools 4** plugin installed

### Steps

1. **Import the project**
   - `File → Import → Maven → Existing Maven Projects`
   - Select the `airbnb-booking` folder → Finish
   - Eclipse will download all dependencies automatically (first run takes 1-2 min)

2. **Run the application**
   - Right-click `BookingApplication.java`
   - `Run As → Spring Boot App` (or `Java Application`)

3. **Verify startup**
   - Console should print:
     ```
     Login credentials – HOST:  host1@demo.com  / password123
     Swagger UI  → http://localhost:8080/swagger-ui.html
     H2 Console  → http://localhost:8080/h2-console
     ```

### Useful URLs

| URL | Description |
|-----|-------------|
| http://localhost:8080/swagger-ui.html | Interactive API docs |
| http://localhost:8080/h2-console | H2 database browser |
| http://localhost:8080/api-docs | Raw OpenAPI JSON |

**H2 Console settings**
- JDBC URL: `jdbc:h2:mem:airbnbdb`
- Username: `sa`
- Password: *(leave blank)*

---

## Quick-Start: Testing the APIs

### Step 1 – Register users

```http
POST /api/auth/register
{
  "name": "My Host",
  "email": "myhost@test.com",
  "password": "pass123",
  "role": "HOST"
}
```

```http
POST /api/auth/register
{
  "name": "My Guest",
  "email": "myguest@test.com",
  "password": "pass123",
  "role": "GUEST"
}
```

### Step 2 – Login and copy the token

```http
POST /api/auth/login
{
  "email": "myhost@test.com",
  "password": "pass123"
}
```

Response contains `"token": "eyJ..."` – copy it.

### Step 3 – Use the token in all secured requests

Header: `Authorization: Bearer <paste-token-here>`

### Step 4 – Create a property (HOST)

```http
POST /api/properties
Authorization: Bearer <host-token>

{
  "title": "Beach Villa",
  "description": "Beautiful sea-facing villa",
  "location": "Goa",
  "pricePerNight": 5000.00
}
```

### Step 5 – Set availability (HOST)

```http
POST /api/properties/1/availability
Authorization: Bearer <host-token>

{
  "availableFrom": "2026-06-10",
  "availableTo":   "2026-12-31"
}
```

### Step 6 – Search properties (public)

```http
GET /api/properties?location=Goa
GET /api/properties?location=Hyderabad&minPrice=1000&maxPrice=5000
GET /api/properties/popular
```

### Step 7 – Book a property (GUEST)

```http
POST /api/bookings
Authorization: Bearer <guest-token>

{
  "propertyId": 1,
  "startDate":  "2026-06-15",
  "endDate":    "2026-06-18"
}
```

### Step 8 – Booking lifecycle (HOST)

```http
PUT /api/bookings/1/confirm    → CONFIRMED
PUT /api/bookings/1/complete   → COMPLETED
PUT /api/bookings/1/cancel     → CANCELLED
```

### Step 9 – View booking history

```http
GET /api/bookings/user/{userId}
GET /api/bookings/property/{propertyId}   (HOST only)
```

### Step 10 – Add a review (GUEST – only after COMPLETED stay)

```http
POST /api/reviews
Authorization: Bearer <guest-token>

{
  "propertyId": 1,
  "rating": 5,
  "comment": "Amazing place, will come again!"
}
```

---

## Key Business Rules

| Rule | Implementation |
|------|----------------|
| No double-booking | `BookingRepository.hasOverlappingBooking` – SQL date-overlap check |
| Availability required | `PropertyAvailabilityRepository.isPropertyAvailableForDates` |
| Review after stay only | `BookingRepository.hasCompletedBooking` – checks COMPLETED status |
| One review per guest | `ReviewRepository.existsByPropertyIdAndGuestId` |
| Total price auto-calc | nights × pricePerNight at booking time |
| Role enforcement | `@AuthenticationPrincipal` + service-layer role checks |

---

## Booking Status Flow

```
REQUESTED → CONFIRMED → COMPLETED
    ↓            ↓
CANCELLED    CANCELLED
```

---

## API Endpoints Summary

### Auth
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/register` | Register HOST or GUEST |
| POST | `/api/auth/login` | Login – returns JWT |

### Properties
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/properties` | HOST | Create listing |
| PUT | `/api/properties/{id}` | HOST | Update listing |
| GET | `/api/properties` | Public | Search by location / price |
| GET | `/api/properties/{id}` | Public | Property details |
| POST | `/api/properties/{id}/availability` | HOST | Set availability |
| GET | `/api/properties/host/{hostId}` | Public | All properties by host |
| GET | `/api/properties/popular` | Public | Popular properties |
| GET | `/api/properties/{id}/statistics` | HOST | Booking statistics |

### Bookings
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/bookings` | GUEST | Create booking |
| GET | `/api/bookings/user/{userId}` | Auth | Booking history |
| GET | `/api/bookings/property/{propertyId}` | HOST | Property bookings |
| PUT | `/api/bookings/{id}/cancel` | Auth | Cancel booking |
| PUT | `/api/bookings/{id}/confirm` | HOST | Confirm booking |
| PUT | `/api/bookings/{id}/complete` | HOST | Complete booking |

### Reviews
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/reviews` | GUEST | Add review (after stay) |
| GET | `/api/reviews/property/{propertyId}` | Public | Property reviews |

---

## Switching to MySQL / PostgreSQL

1. Add the driver to `pom.xml`:

```xml
<!-- MySQL -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

2. In `application.properties`, comment out the H2 block and uncomment:

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/airbnbdb?createDatabaseIfNotExist=true&useSSL=false&serverTimezone=UTC
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.username=root
spring.datasource.password=your_password
spring.jpa.database-platform=org.hibernate.dialect.MySQLDialect
spring.jpa.hibernate.ddl-auto=update
```

3. Restart – Hibernate creates all tables automatically.

---

## Demo Accounts (seeded on startup)

| Role | Email | Password |
|------|-------|----------|
| HOST | host1@demo.com | password123 |
| HOST | host2@demo.com | password123 |
| GUEST | guest1@demo.com | password123 |
| GUEST | guest2@demo.com | password123 |
