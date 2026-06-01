# Backend Readme on functionality


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
- Returns whatever the service gives back as JSON no transformation

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

### Entity — `DeveloperMetric.java`
- Represents one full row of developer activity for one day
- Fields: `developerName`, `metricDate`, `commits`, `bugsFixed`, `tasksCompleted`, `storyPoints`
- This is the internal "full record" it knows everything about one data point
- It is never persisted to a database; it is a plain Java object

---

## DTOs

### `MetricResponseDTO` (used in `dto/` package)
- This is what the frontend actually receives
- Has only 2 fields `label` (the date as a string) and `value` (the metric number)
- The service converts each full `DeveloperMetric` entity into this DTO
- Its purpose is to hide all internal fields and only expose what the chart needs

### `MetricRequestDTO` (unused — in `dto/` package)
- Has one field: `metric` (a string)
- Was likely intended to receive the metric type in a POST request body
- **Never used anywhere in the code**
- The metric type is passed through the URL path instead (`/metrics/commits`)
- Dead code

---

## Inconsistencies found

### 1. Duplicate `MetricResponseDTO`
There are two identical classes with the same name and fields (`label`, `value`):
- `com.exampleback.demo.dto.MetricResponseDTO` this is the one actually used
- `com.exampleback.demo.repository.MetricResponseDTO` this one is never referenced

The one inside the `repository` package is a leftover copy and should be deleted.

### 2. `MetricRequestDTO` is never used
The class `dto/MetricRequestDTO.java` exists but is never injected, referenced, or consumed anywhere. The controller receives the metric type from the URL path variable, making this DTO completely redundant.

### 3. No real database despite JPA and H2 being declared (hardcoded)

### 4. Firebase Admin Hardcoded

### 5. `application.properties` is nearly empty
Only `spring.application.name=demo` is set. No port, no DB config, no environment specific settings.

---

