# Subscription Master

A production-ready **Subscription Management Backend API** built with **Java 21**, **Spring Boot 3**, and **PostgreSQL**.

Subscription Master helps users track, manage, and optimize all recurring paid subscriptions (e.g., Netflix, Spotify, ChatGPT Plus, YouTube Premium, Adobe Creative Cloud, JetBrains, AWS, etc.) in one unified platform. It automatically calculates next payment dates, converts multi-currency expenses to **UZS** using live **Central Bank of Uzbekistan (CBU)** rates, sends payment-due reminders, provides interactive calendar dynamics, and exports annual financial reports.

The project fulfills all requirements of the **Technical Task / Take-Home Assignment (Junior / Junior+ Backend Developer)**, with additional enhancements for frontend integration, resilient external communication, security rate limiting, and automated CI/CD deployment on AWS EC2.

---

## 🌐 Live Production Links

| Resource | URL |
| :--- | :--- |
| **Production Domain** | `https://subscriptionmaster.app` |
| **Swagger UI (Interactive API Docs)** | [https://subscriptionmaster.app/api/swagger-ui/index.html](https://subscriptionmaster.app/api/swagger-ui/index.html) |
| **OpenAPI v3 JSON Specification** | [https://subscriptionmaster.app/api/v3/api-docs](https://subscriptionmaster.app/api/v3/api-docs) |
| **Health Check Endpoint** | [https://subscriptionmaster.app/api/actuator/health](https://subscriptionmaster.app/api/actuator/health) |

> **Note on URL Structure:** The application uses `server.servlet.context-path: /api`. All endpoints, including Swagger and Actuator, are served under the `/api` prefix.

---

## 🎯 Problem & Objectives

Modern digital consumers maintain numerous subscriptions billed across various cycles (monthly, annual, weekly) and different currencies (`USD`, `EUR`, `UZS`). Users frequently struggle with:
1. **Financial blindness**: Losing track of total monthly and annual commitments.
2. **Renewal surprises**: Missing renewal deadlines and incurring unexpected credit card charges.
3. **Currency fluctuations**: Not knowing the actual cost in their local base currency (`UZS`) as exchange rates shift.
4. **Scattered history**: Losing payment and transaction records once a subscription is canceled or paused.

**Subscription Master solves this by providing:**
* Single-pane subscription tracking with status management (`ACTIVE`, `PAUSED`, `CANCELLED`).
* Multi-currency normalization to base currency (`UZS`) via official CBU rates.
* Automatic payment simulation and permanent payment history logging.
* Automated daily payment reminders (via Log and Gmail).
* Granular filtering, search, pagination, and visual calendar data.
* Spending dynamics and exportable CSV / Excel (`.xlsx`) annual reports.

---

## ✅ Implemented Features (What Is Done & What Is Added)

### 1. Secure Authentication & User Management (Spring Security + JWT)
* **Registration & Login**: `POST /api/auth/register` and `POST /api/auth/login`. Passwords hashed using **BCrypt** (`strength = 10`).
* **Stateless JWT**: Access tokens (15-minute validity) signed with HMAC-SHA256 (`jjwt` 0.12.6).
* **Refresh Token Rotation**: `POST /api/auth/refresh` issues a new access token without re-entering credentials; stored in PostgreSQL and validated per request.
* **Token Revocation & Logout**: `POST /api/auth/logout` deletes active refresh tokens on logout.
* **Current Profile**: `GET /api/auth/me` returns the authenticated user's ID, username, role, and authorities.
* **Brute-Force Rate Limiting**: In-memory sliding window using Caffeine (`AuthRateLimiter`). Blocks an IP address for 60 seconds after 3 consecutive failed login attempts, responding with HTTP `429 Too Many Requests` and headers `Retry-After`, `X-RateLimit-Retry-After-Seconds`, and `X-RateLimit-Blocked-Until`.
* **Resource Isolation & Ownership**: Authorization-level check ensures users can only read, update, pause, resume, or delete their own subscriptions.
* **Role-Based Access Control**:
  * `USER`: Full management of own subscriptions, reports, and statistics.
  * `ADMIN`: Access to platform-wide statistics and service usage.

### 2. Subscription Management (Advanced CRUD)
* **Full Lifecycle**: `POST`, `GET`, `PUT`, `DELETE` operations on `/api/subscriptions`.
* **Supported Currencies**: `USD`, `EUR`, `UZS`.
* **Billing Frequencies**: `WEEKLY`, `MONTHLY`, `ANNUAL`.
* **Statuses**: `ACTIVE`, `PAUSED`, `CANCELLED` (Java `enum`).
* **Categories**: `ENTERTAINMENT`, `PRODUCTIVITY`, `AI_TOOLS`, `CLOUD_SERVICES`, `UTILITIES`, `EDUCATION`, `GAMING`, `OTHER`.
* **Automatic Next Payment Date Calculation**: Computes `nextPaymentDate` on creation and updates it based on start date and frequency.
* **Pause & Resume Workflow**:
  * `POST /api/subscriptions/{id}/pause`: Safely pauses an active subscription.
  * `POST /api/subscriptions/{id}/resume`: Resumes a paused subscription.
* **Soft Delete**: When a subscription is deleted (`DELETE /api/subscriptions/{id}`), it is marked with `isDeleted = true` instead of physical deletion, keeping foreign-key integrity and payment records intact.
* **Payment History Retention**: `GET /api/subscriptions/{id}/history` returns all past charges (date, original amount/currency, converted amount in UZS, and exchange rate applied) even if the subscription is paused or canceled.
* **Upcoming Renewals**: `GET /api/subscriptions/upcoming?days=7` lists subscriptions due for renewal within the next N days.
* **Renewal Calendar**: `GET /api/subscriptions/calendar?year=2026&month=10` groups subscriptions by exact payment date within the month and calculates total daily and monthly expenses converted to UZS.
* **Unpaged Export List**: `GET /api/subscriptions/all` returns all filtered subscriptions without pagination for frontends rendering calendar views or dropdowns.

### 3. Advanced Filtering & Pagination (JPA Criteria / Specification)
* **Zero Native SQL**: Built entirely using Spring Data JPA `Specification<Subscription>` and Criteria API.
* **Combined Filters**:
  * Status (`status=ACTIVE`)
  * Currency (`currency=USD`)
  * Category (`category=AI_TOOLS`, case-insensitive)
  * Name search (`name=net`, case-insensitive `LIKE %name%`)
  * Renewal date range (`nextPaymentDateFrom` and `nextPaymentDateTo`)
  * Multi-currency Price Range (`minPrice` and `maxPrice`): Automatically converts subscriptions in foreign currencies to UZS using live exchange rates so filtering accurately compares costs across mixed currencies.
* **Flexible Pageable & Sort Query Syntax**: Custom argument resolvers supporting `sort=price,desc`, `sort=nextPaymentDate,asc`, and unpaged queries (`unpaged=true` or `size=-1`).

### 4. External Currency Conversion & Resilience (CBU Integration)
* **Open API Client**: `CbuRateClient` queries the official Central Bank of Uzbekistan API (`https://cbu.uz/uz/arkhiv-kursov-valyut/json/`) using Spring's modern non-blocking `RestClient`.
* **Caching**: Spring Cache with **Caffeine** (`cache-names: exchange-rates`, `expireAfterWrite: 24h`) ensures external API calls occur only once daily, not per request.
* **Resilience4j Circuit Breaker**: Wraps CBU API requests with `@CircuitBreaker(name = "cbuRates", fallbackMethod = "unavailable")`. If the external API times out or fails (5xx), the circuit opens and falls back to the most recent cached rate in the `exchange_rate` database table.
* **Cold-Start Fallbacks**: If both external API and local cache are unavailable, safe static defaults (`USD: 12850.00`, `EUR: 13967.50`, `UZS: 1.00`) prevent system downtime.
* **Authoritative Exchange Rates API**: `GET /api/currencies/rates` exposes live base-currency rates and the exact timestamp (`updatedAt`).

### 5. Automated Scheduling & Concurrency (ShedLock)
* **Multi-Instance Safe (ShedLock)**: Prevents duplicate job execution when horizontally scaled across multiple containers using the PostgreSQL `shedlock` table (`@SchedulerLock`).
* **Scheduled Tasks (`PaymentService`)**:
  * **09:00 Tashkent (`0 0 9 * * *`)**: Checks the database for subscriptions due in exactly 2 days (`nextPaymentDate = now + 2`) and triggers notifications.
  * **00:05 Tashkent (`0 5 0 * * *`)**: Identifies past-due subscriptions, writes simulation entries to `payment_history`, and advances `nextPaymentDate` to the next billing cycle.
  * **00:10 Tashkent (`0 10 0 * * *`)**: Refreshes daily currency rates from CBU into the database cache.

### 6. Notification Strategy (Strategy Pattern)
* Pluggable notification channel abstraction via `NotificationService` interface:
  * `LogNotificationService`: Structured logging of upcoming payment warnings.
  * `GmailNotifier`: JavaMailSender implementation sending HTML/text email warnings via Gmail SMTP with TLS.

### 7. Financial Reports Export (File I/O)
* **CSV Export**: `GET /api/reports/annual.csv` generates downloadable UTF-8 CSV with columns: `Name, Monthly Price, Currency, Monthly Equivalent (UZS), Total Annual Cost (UZS)`.
* **Excel Export**: `GET /api/reports/annual.xlsx` generates Microsoft Excel spreadsheets using **Apache POI** (`poi-ooxml` 5.3.0) with styled headers, auto-sized columns, and formula totals.

### 8. Analytics & Admin Statistics
* **User Spending Summary**: `GET /api/statistics/summary` returns:
  * Total monthly spending in UZS (`monthlyCost`)
  * Most expensive subscription (`mostExpensive`)
  * Spending breakdown grouped by category (`CategoryCost`)
* **Monthly Dynamics**: `GET /api/statistics/monthly-dynamics?months=6` returns monthly spending trends for the last 6 or 12 months, formatted as JSON arrays ready for charts.
* **Admin Usage Ranking**: `GET /api/admin/statistics/service-usage` provides platform-wide service popularity counts (admin-only).

### 9. Database & Seed Data
* **Migrations via Flyway**: Versioned SQL scripts in `src/main/resources/db/migration/` managing relational schema, foreign keys, and indexes.
* **Realistic Seed Data**: `DataLoader` automatically populates the database on startup (configurable via `APP_SEED_ENABLED=true`) with **200 realistic subscriptions** across categories, currencies, and users.

---

## 🏗 Architectural Principles & Clean Code

| Requirement | Implementation Detail |
| :--- | :--- |
| **No Lombok** | Pure modern Java: `record` classes for DTOs, explicit constructors, and encapsulation. |
| **No Field Injection** | 100% **Constructor Injection** across all `@Service`, `@RestController`, and `@Component` classes. |
| **No Eager Fetching** | All entity relationships use `FetchType.LAZY` to prevent N+1 queries. |
| **No Native SQL** | Dynamic queries use JPA `Specification` and Criteria API; database operations use standard Spring Data JPA methods. |
| **Optimistic Locking** | `@Version private Long version;` on `Subscription` and `User` prevents concurrent update collisions. |
| **Auditing** | Base class `AuditableEntity` tracks `@CreatedDate` and `@LastModifiedDate` using Spring Data JPA auditing. |
| **Global Exception Handling** | `@RestControllerAdvice` (`GlobalExceptionHandler`) intercepts `NotFoundException`, `ConflictException`, `ForbiddenException`, validation errors, and rate limits, returning standardized `ErrorResponse` JSON. |
| **Transaction Integrity** | Read-only transactions by default on services (`@Transactional(readOnly = true)`); write operations specify `@Transactional` with automatic rollback on runtime errors. |

---

## 🛠 Tech Stack

* **Language**: Java 21 LTS
* **Framework**: Spring Boot 3.4.1
* **Security**: Spring Security 6, JJWT (0.12.6), BCrypt
* **Database**: PostgreSQL 16
* **Database Migrations**: Flyway
* **Object Mapping**: MapStruct 1.6.3
* **Caching**: Spring Cache, Caffeine
* **Resilience**: Resilience4j (Circuit Breaker)
* **Distributed Locking**: ShedLock 6.2.0 (PostgreSQL provider)
* **Reporting**: Apache POI 5.3.0 (`poi-ooxml`)
* **API Documentation**: Springdoc OpenAPI / Swagger UI 2.8.3
* **Containerization**: Docker, Docker Compose, Jib Maven Plugin
* **Reverse Proxy**: Nginx with TLS (HTTPS)
* **CI/CD**: GitHub Actions deploying to AWS EC2

---

## 📚 Complete API Reference (All 20 Endpoints)

All endpoints are prefixed with `/api`.

### 🔑 Authentication (`/api/auth`)
| Method | Endpoint | Access | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/auth/register` | Public | Register new user account |
| `POST` | `/api/auth/login` | Public | Authenticate user; returns access and refresh tokens |
| `POST` | `/api/auth/refresh` | Public | Issue a new access token from a refresh token |
| `POST` | `/api/auth/logout` | Public | Revoke active refresh token |
| `GET` | `/api/auth/me` | Authenticated | Get current authenticated user profile |

### 💳 Subscriptions (`/api/subscriptions`)
| Method | Endpoint | Access | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/subscriptions` | `USER` / `ADMIN` | Create a new subscription |
| `GET` | `/api/subscriptions` | `USER` / `ADMIN` | Paginated and filtered list of subscriptions |
| `GET` | `/api/subscriptions/all` | `USER` / `ADMIN` | Unpaged list with identical filter support |
| `GET` | `/api/subscriptions/{id}` | `USER` / `ADMIN` | Get subscription details by ID (own only) |
| `PUT` | `/api/subscriptions/{id}` | `USER` / `ADMIN` | Update an existing subscription |
| `DELETE` | `/api/subscriptions/{id}` | `USER` / `ADMIN` | Soft delete subscription (`isDeleted = true`) |
| `POST` | `/api/subscriptions/{id}/pause` | `USER` / `ADMIN` | Pause an active subscription |
| `POST` | `/api/subscriptions/{id}/resume` | `USER` / `ADMIN` | Resume a paused subscription |
| `GET` | `/api/subscriptions/{id}/history` | `USER` / `ADMIN` | View payment history for subscription |
| `GET` | `/api/subscriptions/upcoming` | `USER` / `ADMIN` | Subscriptions due for renewal within next N days |
| `GET` | `/api/subscriptions/calendar` | `USER` / `ADMIN` | Calendar grouped by day with UZS totals |

### 📊 Statistics (`/api/statistics`)
| Method | Endpoint | Access | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/statistics/summary` | `USER` / `ADMIN` | Monthly cost, most expensive, category breakdown |
| `GET` | `/api/statistics/monthly-dynamics` | `USER` / `ADMIN` | Historical monthly spending trends (6 or 12 months) |

### 💱 Currencies (`/api/currencies`)
| Method | Endpoint | Access | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/currencies/rates` | Public | Current CBU exchange rates to UZS |

### 📁 Reports (`/api/reports`)
| Method | Endpoint | Access | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/reports/annual.csv` | `USER` / `ADMIN` | Export annual cost analysis in CSV |
| `GET` | `/api/reports/annual.xlsx` | `USER` / `ADMIN` | Export annual cost analysis in Excel (`.xlsx`) |

### 👑 Admin (`/api/admin`)
| Method | Endpoint | Access | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/admin/statistics/service-usage` | `ADMIN` | Platform-wide service popularity counts |

---

## 🚀 Getting Started & How to Use

### Option 1: Test via Live Swagger UI

1. Open [https://subscriptionmaster.app/api/swagger-ui/index.html](https://subscriptionmaster.app/api/swagger-ui/index.html).
2. Register an account under **`auth-controller` ➔ `POST /auth/register`**:
   ```json
   {
     "username": "developer",
     "email": "dev@example.com",
     "password": "Password123!"
   }
   ```
3. Copy the returned `accessToken`.
4. Click the green **Authorize** button at the top-right of Swagger UI.
5. Paste the token into the value field and click **Authorize**.
6. Create a subscription via **`POST /subscriptions`**:
   ```json
   {
     "name": "ChatGPT Plus",
     "price": 20.00,
     "currency": "USD",
     "frequency": "MONTHLY",
     "status": "ACTIVE",
     "category": "AI_TOOLS",
     "startDate": "2026-10-01"
   }
   ```
7. Explore:
   * View paginated subscriptions: `GET /subscriptions`
   * Check upcoming renewals: `GET /subscriptions/upcoming?days=30`
   * View renewal calendar: `GET /subscriptions/calendar?year=2026&month=10`
   * Get spending statistics: `GET /statistics/summary`
   * Download CSV or Excel reports: `GET /reports/annual.xlsx`

---

### Option 2: Run Locally via Docker Compose

#### Prerequisites
* [Docker Desktop](https://www.docker.com/products/docker-desktop/) (Docker 24+ & Docker Compose v2)
* Port `8080` (or `80`/`443`) and `5222` available.

#### Steps
1. **Clone the repository**:
   ```bash
   git clone https://github.com/mirkamol-code/sub-master.git
   cd sub-master
   ```

2. **Set environment variables**:
   Create a `.env` file in the project root:
   ```env
   POSTGRES_DB=subscription_master_db
   POSTGRES_USER=mirkamol
   POSTGRES_PASSWORD=password
   JWT_SECRET=super-secret-key-that-is-at-least-32-bytes-long!
   APP_SEED_ENABLED=true
   ```

3. **Start services**:
   ```bash
   docker compose up --build -d
   ```

4. **Verify**:
   * API: `http://localhost:8080/api/actuator/health`
   * Swagger UI: `http://localhost:8080/api/swagger-ui/index.html`
   * PostgreSQL port: `localhost:5222`

---

### Option 3: Run Locally for Development (Maven + Local DB)

If you wish to run the Spring Boot app directly inside IntelliJ IDEA or terminal while using PostgreSQL in Docker:

1. **Start only PostgreSQL via Docker**:
   ```bash
   docker compose -f docker-compose-db-only.yml up -d
   ```
   *(Runs PostgreSQL on `localhost:5222`)*

2. **Run Spring Boot**:
   ```bash
   ./mvnw spring-boot:run
   ```
   *(Or run `SubscribeMasterApplication.java` from IntelliJ IDEA)*

3. **Run All Tests**:
   ```bash
   mvn clean verify
   ```
   *Uses Testcontainers (`SharedPostgresContainer`) to run integration tests against real PostgreSQL instances.*

---

## ⚙️ Key Configuration Properties

| Property | Default / Example | Purpose |
| :--- | :--- | :--- |
| `server.servlet.context-path` | `/api` | Root context path for all controllers and tools |
| `spring.datasource.url` | `jdbc:postgresql://localhost:5222/subscription_master_db` | PostgreSQL connection URL |
| `app.jwt.secret` | `secret-key-must-be-at-least-32-bytes` | HMAC-SHA256 signing secret for JWTs |
| `app.jwt.access-token-minutes` | `15` | JWT expiration duration |
| `app.jwt.refresh-token-days` | `30` | Refresh token expiration duration |
| `app.rate-limiting.auth.max-failed-attempts` | `3` | Number of bad logins before blocking IP |
| `app.rate-limiting.auth.block-duration-seconds` | `60` | Duration of brute-force lock |
| `app.seed.enabled` | `true` | Auto-seed 200 realistic subscriptions |
| `app.cbu.base-url` | `https://cbu.uz/uz/arkhiv-kursov-valyut/json/` | Central Bank of Uzbekistan open rate feed |

---

## 🧪 Testing Coverage

The project maintains comprehensive test coverage across unit and integration layers (25 test classes):
* **Unit Tests**:
  * `SubscriptionServiceTest`, `AuthServiceTest`, `ExchangeRateServiceTest`, `StatisticServiceTest`, `ReportServiceTest`, `PaymentServiceTest`, `NotificationServiceTest`.
  * `AuthRateLimiterTest`: Verifies failed attempt tracking, lockout timing, and counter reset on success.
* **Integration Tests (Testcontainers + PostgreSQL)**:
  * `SubscriptionControllerIT`, `AuthControllerIT`, `CurrencyControllerIT`, `ReportControllerIT`, `StatisticControllerIT`, `AdminControllerIT`.
  * Verifies security filters, JWT validation, database rollbacks, ShedLock tables, and HTTP status codes.

Run tests locally with:
```bash
mvn test
```

---

## 📄 License & Attribution

Developed as a demonstration of clean enterprise Java engineering for the **Subscription Master Take-Home Assignment**.
All rights reserved © 2026.
