# BookMyTurf – Turf & Venue Booking System 

A complete Spring Boot backend for booking cricket/sports turfs online, with JWT auth
and a hybrid, rule-based turf recommendation engine.

## Tech stack
- Java 17
- Spring Boot 3.2.5 (Web, Data JPA, Security, Validation)
- MySQL 8
- JWT (jjwt 0.12.5)
- Maven
- Lombok

## 1. Prerequisites
- JDK 17+
- Maven 3.8+ (or use an IDE that bundles it — IntelliJ, VS Code, Eclipse)
- MySQL 8 running locally (or update the connection string to point elsewhere)

## 2. Configure the database
Open `src/main/resources/application.properties` and set your MySQL credentials:

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/bookmyturf_db?createDatabaseIfNotExist=true&useSSL=false&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=root
```

The database `bookmyturf_db` is created automatically on first run
(`createDatabaseIfNotExist=true`), and tables are created/updated automatically via
`spring.jpa.hibernate.ddl-auto=update`. You don't need to run any SQL scripts by hand.

## 3. Run it

```bash
# from the project root (where pom.xml lives)
mvn spring-boot:run
```

Or build a jar and run it:

```bash
mvn clean package -DskipTests
java -jar target/bookmyturf-1.0.0.jar
```

The API starts on **http://localhost:8080**.

> If you use IntelliJ/Eclipse/VS Code, just import it as a Maven project and run
> `BookMyTurfApplication.java` directly — no CLI needed.

## 4. Project structure

```
src/main/java/com/bookmyturf
├── controller        REST endpoints
├── service            business logic
├── repository         Spring Data JPA repositories
├── entity              JPA entities (User, Turf, Slot, Booking, Review)
├── dto                  request/response payloads
├── security            JWT filter, JWT service, Spring Security config
└── exception           global exception handling
```

## 5. API reference

### Auth (public)
| Method | Endpoint             | Body                                            |
|--------|-----------------------|--------------------------------------------------|
| POST   | `/api/auth/register`  | `{ "name", "email", "password", "role" }` (role optional: USER/OWNER) |
| POST   | `/api/auth/login`     | `{ "email", "password" }`                        |

Both return `{ token, userId, name, email, role }`. Send the token on every
subsequent request as `Authorization: Bearer <token>`.

### Users
| Method | Endpoint         | Auth |
|--------|-------------------|------|
| GET    | `/api/users/me`   | required |

### Turfs
| Method | Endpoint               | Auth | Notes |
|--------|-------------------------|------|-------|
| POST   | `/api/turfs`            | required | creates a turf owned by the logged-in user |
| GET    | `/api/turfs`            | public | optional `?sport=&location=` query filters |
| GET    | `/api/turfs/{id}`       | public | |
| PUT    | `/api/turfs/{id}`       | required | |
| DELETE | `/api/turfs/{id}`       | required | |

### Slots
| Method | Endpoint                              | Auth |
|--------|-----------------------------------------|------|
| POST   | `/api/turfs/{turfId}/slots`             | required |
| GET    | `/api/turfs/{turfId}/slots`             | public — optional `?date=YYYY-MM-DD` |

### Bookings
| Method | Endpoint                     | Auth |
|--------|-------------------------------|------|
| POST   | `/api/bookings`               | required — `{ "turfId", "slotId" }` |
| GET    | `/api/bookings/my`            | required |
| PUT    | `/api/bookings/{id}/cancel`   | required |

### Reviews
| Method | Endpoint                       | Auth |
|--------|----------------------------------|------|
| POST   | `/api/reviews`                  | required — `{ "turfId", "rating" (1-5), "comment" }` |
| GET    | `/api/reviews/turf/{turfId}`    | public |

A turf's average rating is recalculated automatically whenever a review is added.

### Recommendations
| Method | Endpoint              | Auth |
|--------|------------------------|------|
| POST   | `/api/recommendations` | public |

Request:
```json
{
  "sport": "CRICKET",
  "location": "Nashik",
  "budget": 1500,
  "players": 10,
  "date": "2026-09-20",
  "preferredTime": "19:00:00"
}
```

Response:
```json
{
  "recommendations": [
    {
      "turfId": 3,
      "turf": "Green Arena",
      "matchScore": 92.0,
      "reason": "Matches your preferred sport, in your preferred location, within your budget, highly rated by other players."
    }
  ]
}
```

## 6. How the recommendation engine works

This is the "hybrid" approach from the project plan — no ML training required for v1:

```
Database filtering (sport, availability)
        +
User preferences (location, budget, time)
        +
Turf rating
        ↓
Weighted score (0–100) + ranked list
```

It's implemented in `RecommendationService.java`. Because it's isolated behind a clean
`recommend(RecommendationRequest)` method, you can later swap in a trained ML model or an
LLM call (e.g. via the Anthropic/OpenAI API) without touching the controller or the rest
of the app — exactly the "upgrade path" mentioned in the original project plan.

## 7. Quick test flow (with curl)

```bash
# 1. Register
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Rahul","email":"rahul@example.com","password":"password123","role":"OWNER"}'

# copy the "token" from the response, then:
TOKEN="paste-token-here"

# 2. Create a turf
curl -X POST http://localhost:8080/api/turfs \
  -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" \
  -d '{"name":"Green Arena","location":"Nashik","sport":"CRICKET","price":1200,"description":"6-a-side turf"}'

# 3. Add a slot for turf id 1
curl -X POST http://localhost:8080/api/turfs/1/slots \
  -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" \
  -d '{"slotDate":"2026-09-20","startTime":"19:00:00","endTime":"20:00:00"}'

# 4. Book it
curl -X POST http://localhost:8080/api/bookings \
  -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" \
  -d '{"turfId":1,"slotId":1}'

# 5. Get recommendations
curl -X POST http://localhost:8080/api/recommendations \
  -H "Content-Type: application/json" \
  -d '{"sport":"CRICKET","location":"Nashik","budget":1500,"date":"2026-09-20","preferredTime":"19:00:00"}'
```

## 8. Notes / next steps
- Postman: import the endpoints above into a collection to test manually.
- The `role` field lets a user register as `OWNER` (can list turfs) or plain `USER`
  (books turfs). `ADMIN` exists as an enum value for future use, but no admin-only
  guard is enforced yet — add `@PreAuthorize("hasRole('ADMIN')")` where needed.
- Passwords are hashed with BCrypt and never returned in API responses.
- `spring.jpa.hibernate.ddl-auto=update` is convenient for development; switch to a
  migration tool (Flyway/Liquibase) before production use.
