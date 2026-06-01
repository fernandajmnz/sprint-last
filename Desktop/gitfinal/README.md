# sprint-last — Backend Documentation

Spring Boot REST API that serves developer productivity metrics to a frontend chart application.

---

## How it works

The backend exposes a single endpoint. When the frontend calls it with a metric type, the backend fetches hardcoded records, picks the right column, and returns a simplified list of `{ label, value }` pairs ready for charting.

```
GET /metrics/{metric}
         ↓
Controller extracts the metric name from the URL
         ↓
Service fetches all 4 hardcoded DeveloperMetric records
         ↓
For each record → builds { label: date, value: <chosen metric> }
         ↓
Returns JSON array to the frontend
```

Valid values for `{metric}`: `commits`, `bugs`, `tasks`, `storyPoints`

---

## Layers

### Entry Point — `DemoApplication.java`
Bootstraps the Spring context and starts the embedded Tomcat server. Nothing custom here.

---

### Controller — `MetricsController.java`
- Listens on `GET /metrics/{metric}`
- Reads the `{metric}` path variable from the URL
- Passes it directly to the service
- Returns whatever the service gives back as JSON — no transformation

---

### Service — `MetricsService.java`
This is where the core logic lives.
- Calls the repository to get all records
- Loops over them and for each one:
  - Sets `label` = the date of that record (as a string)
  - Uses a `switch` on the metric name to decide which number becomes `value`
  - `commits` → commits field
  - `bugs` → bugsFixed field
  - `tasks` → tasksCompleted field
  - `storyPoints` → storyPoints field
  - Any unknown metric → value defaults to `0`
- Returns a list of `MetricResponseDTO` objects

---

### Repository — `DeveloperMetricRepository.java`
- Normally this layer talks to a database — here it just returns 4 hardcoded objects
- All 4 records are for developer **"Francisco"**, spanning **May 1–4, 2026**
- Each record has: commits, bugsFixed, tasksCompleted, storyPoints
- This is the only source of data in the entire app

---

### Entity — `DeveloperMetric.java`
- Represents one full row of developer activity for one day
- Fields: `developerName`, `metricDate`, `commits`, `bugsFixed`, `tasksCompleted`, `storyPoints`
- This is the internal "full record" — it knows everything about one data point
- It is never persisted to a database; it is a plain Java object

---

## DTOs

### `MetricResponseDTO` (used — in `dto/` package)
- This is what the frontend actually receives
- Has only 2 fields: `label` (the date as a string) and `value` (the metric number)
- The service converts each full `DeveloperMetric` entity into this DTO
- Example: a record with commits=12, bugs=2, tasks=5, storyPoints=8 becomes `{ label: "2026-05-01", value: 12 }` when requesting `commits`
- Its purpose is to hide all internal fields and only expose what the chart needs

### `MetricRequestDTO` (unused — in `dto/` package)
- Has one field: `metric` (a string)
- Was likely intended to receive the metric type in a POST request body
- **Never used anywhere in the code**
- The metric type is passed through the URL path instead (`/metrics/commits`)
- Dead code

---

## Configuration

### `CorsConfig.java`
- Allows cross-origin requests only from `http://localhost:5173` (Vite dev server)
- Allows methods: GET, POST, PUT, DELETE, OPTIONS
- This is what lets the frontend talk to the backend during local development

### `SecurityConfig.java`
- Disables CSRF protection
- Allows all requests without authentication
- Effectively turns off Spring Security so any call to `/metrics/*` goes through freely

---

## Inconsistencies found

### 1. Duplicate `MetricResponseDTO`
There are two identical classes with the same name and fields (`label`, `value`):
- `com.exampleback.demo.dto.MetricResponseDTO` — this is the one actually used
- `com.exampleback.demo.repository.MetricResponseDTO` — this one is never referenced

The one inside the `repository` package is a leftover copy and should be deleted.

### 2. `MetricRequestDTO` is never used
The class `dto/MetricRequestDTO.java` exists but is never injected, referenced, or consumed anywhere. The controller receives the metric type from the URL path variable, making this DTO completely redundant.

### 3. No real database despite JPA and H2 being declared
`pom.xml` includes `spring-boot-starter-data-jpa` and the H2 in-memory database as dependencies, but neither is used. The `DeveloperMetric` model has no JPA annotations (`@Entity`, `@Id`, etc.) and the repository does not extend any Spring Data interface. The data is fully hardcoded.

### 4. Firebase Admin SDK declared but unused
`pom.xml` includes `firebase-admin 9.1.1` as a dependency, but there is no Firebase initialization, no service account config, and no Firebase usage anywhere in the code.

### 5. Hardcoded data with no filtering by developer
The repository always returns all 4 records for "Francisco" regardless of who is asking. There is no developer filtering, pagination, or date range — the endpoint always returns the same fixed dataset.

### 6. `application.properties` is nearly empty
Only `spring.application.name=demo` is set. No port, no DB config, no environment-specific settings.

---

## Tech stack

| Technology | Version |
|---|---|
| Java | 21 |
| Spring Boot | 4.0.6 |
| Spring Security | included via starter |
| Lombok | latest via BOM |
| H2 (declared, unused) | runtime |
| Firebase Admin (declared, unused) | 9.1.1 |
| Build tool | Maven |
