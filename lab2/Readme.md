# Module 02 Laboratory - Spring Boot Foundations and Core

## Course Catalogue API

---

## 1. Purpose

In this laboratory, you will complete a small HTTP application named **Course Catalogue API**. The starting repository already contains the project structure, domain records, request and response objects, an in-memory repository, exception classes, and focused `TODO` markers.

The objective is not to build a large application. The objective is to make the essential Spring Boot mechanisms visible:

- how a Boot application starts;
- how starter dependencies shape the runtime;
- how the `ApplicationContext` discovers and creates managed objects;
- how constructor injection connects controllers, services, repositories, and configuration;
- how external configuration is bound to a typed object;
- how Spring MVC maps HTTP requests and serializes JSON;
- how validation rejects invalid request bodies;
- how global exception handling turns failures into stable HTTP responses;
- how Actuator exposes application health;
- how an executable JAR can be configured without rebuilding it.

This laboratory intentionally does **not** include a database, security, Docker, or automated tests. Those concerns belong to later modules.

---

## 2. Learning outcomes

After completing the laboratory, you should be able to:

1. Explain the roles of `@SpringBootApplication`, auto-configuration, component scanning, and the application context.
2. Register application components using Spring stereotypes.
3. Connect components with constructor injection.
4. Bind external properties to a `@ConfigurationProperties` record.
5. Implement GET and POST endpoints with Spring MVC annotations.
6. Validate a JSON request with Jakarta Validation.
7. return meaningful `201`, `400`, `404`, and `409` responses.
8. Inspect application health through Spring Boot Actuator.
9. package and run the same application with a configuration override.

---

## 3. Scenario

The application manages a small catalogue of courses. It starts with two courses in memory:

| Code | Title | Credits | Level |
|---|---|---:|---|
| `JAVA-101` | Java Programming Foundations | 6 | `FOUNDATIONAL` |
| `WEB-201` | Web Application Development | 6 | `INTERMEDIATE` |

The completed API must support:

| Operation | Route | Expected success |
|---|---|---|
| Catalogue information | `GET /api/catalog` | `200 OK` |
| List courses | `GET /api/courses` | `200 OK` |
| Find one course | `GET /api/courses/{code}` | `200 OK` |
| Create a course | `POST /api/courses` | `201 Created` |
| Application health | `GET /actuator/health` | `200 OK` with `UP` |

The API must also translate common failures:

| Situation | Expected status |
|---|---:|
| Invalid request body | `400 Bad Request` |
| Unknown course code | `404 Not Found` |
| Duplicate code | `409 Conflict` |
| Catalogue at maximum capacity | `409 Conflict` |

---

## 4. Time plan

Respect the timebox. Stop when the acceptance criteria are satisfied.

| Activity | Time |
|---|---:|
| Start and inspect the project | 5 minutes |
| Bind external configuration | 10 minutes |
| Complete the managed object graph and service | 12 minutes |
| Expose the read endpoints | 15 minutes |
| Add creation, validation, and error translation | 13 minutes |
| Verify health and package the application | 5 minutes |
| **Total** | **60 minutes** |

---

## 5. Starter repository

The starter has the following structure:

```text
course-catalog/
├── README.md
├── pom.xml
├── mvnw
├── mvnw.cmd
├── requests.http
└── src/
    └── main/
        ├── java/edu/catalog/api/
        │   ├── CatalogApplication.java
        │   ├── config/
        │   │   └── CatalogProperties.java
        │   ├── course/
        │   │   ├── CatalogResponse.java
        │   │   ├── Course.java
        │   │   ├── CourseController.java
        │   │   ├── CourseLevel.java
        │   │   ├── CourseRepository.java
        │   │   ├── CourseResponse.java
        │   │   ├── CourseService.java
        │   │   ├── CreateCourseRequest.java
        │   │   └── InMemoryCourseRepository.java
        │   └── error/
        │       ├── ApiError.java
        │       ├── CatalogCapacityException.java
        │       ├── CourseNotFoundException.java
        │       ├── DuplicateCourseException.java
        │       └── GlobalExceptionHandler.java
        └── resources/
            └── application.properties
```

Search the project for `TODO`. Each marker belongs to one of the laboratory tasks.

---

## 6. Task 0 - Start and inspect the application

### 6.1 Run the project

On macOS or Linux:

```bash
./mvnw spring-boot:run
```

On Windows:

```powershell
mvnw.cmd spring-boot:run
```

The first execution downloads Maven and the project dependencies, so it can take longer than later executions.

### 6.2 Verify the baseline

Open another terminal and execute:

```bash
curl -i http://localhost:8080/actuator/health
```

Expected body:

```json
{"status":"UP"}
```

The application starts even though the course routes are not active yet. This is deliberate: the repository, service, controller, and exception handler are ordinary Java objects until you register them as Spring beans.

### 6.3 Inspect before editing

Locate the following evidence:

- the three starter dependencies in `pom.xml`;
- the `main` method and `SpringApplication.run(...)`;
- `@SpringBootApplication`;
- `@ConfigurationPropertiesScan`;
- the embedded server port in the startup log;
- the application name and catalogue properties in `application.properties`.

Do not add dependencies.

---

## 7. Task 1 - Bind external configuration 

Open:

```text
src/main/java/edu/catalog/api/config/CatalogProperties.java
src/main/resources/application.properties
```

The record must represent this configuration namespace:

```properties
catalog.title=Application Development Catalogue
catalog.max-courses=10
```

### Requirements

1. Confirm that `CatalogProperties` uses `@ConfigurationProperties(prefix = "catalog")`.
2. Confirm that configuration-properties scanning is enabled from the root application package.
3. Keep the property names external and the Java components typed:
   - `title` is a `String`;
   - `maxCourses` is an `int`.
4. Restart the application with an environment override.

macOS or Linux:

```bash
CATALOG_TITLE="Local Course Catalogue" ./mvnw spring-boot:run
```

Windows PowerShell:

```powershell
$env:CATALOG_TITLE="Local Course Catalogue"
.\mvnw.cmd spring-boot:run
```

You will observe the changed value after the catalogue endpoint is activated in Task 3.

### Checkpoint

Be prepared to explain why the environment variable is named `CATALOG_TITLE` and why the JAR does not need to be rebuilt.

---

## 8. Task 2 - Complete the beans and service logic 

Open:

```text
src/main/java/edu/catalog/api/course/InMemoryCourseRepository.java
src/main/java/edu/catalog/api/course/CourseService.java
```

### 8.1 Register managed components

Add the appropriate Spring stereotype to:

- `InMemoryCourseRepository`;
- `CourseService`.

Do not instantiate these classes with `new` in the controller or application class. Spring must create them and satisfy their constructors.

### 8.2 Complete `CourseService.findAll()`

Return all repository courses as `CourseResponse` values. The repository already sorts by code.

Useful operations:

```java
repository.findAll()
CourseResponse.from(course)
```

### 8.3 Complete `CourseService.findByCode(...)`

The method must:

1. trim and uppercase the supplied code;
2. ask the repository for the normalized code;
3. convert the course to `CourseResponse`;
4. throw `CourseNotFoundException` when the course is absent.

A caller using `java-101`, ` JAVA-101 `, or `JAVA-101` must address the same course.

### 8.4 Complete `CourseService.create(...)`

The method must:

1. normalize the requested code;
2. throw `DuplicateCourseException` when the code already exists;
3. throw `CatalogCapacityException` when `repository.count()` has reached `properties.maxCourses()`;
4. construct a `Course` from the request;
5. save it;
6. return `CourseResponse.from(savedCourse)`.

### Checkpoint

Sketch the object graph:

```text
CourseController -> CourseService -> CourseRepository
                         |
                         +-> CatalogProperties
```

Who creates each object after the stereotypes are added?

---

## 9. Task 3 - Expose the read endpoints 

Open:

```text
src/main/java/edu/catalog/api/course/CourseController.java
```

### 9.1 Register the HTTP adapter

Make the class a REST controller and give it the common prefix:

```text
/api
```

Use constructor injection. Do not request the `ApplicationContext` from inside the controller.

### 9.2 Map the routes

Map the existing methods to:

```text
GET /api/catalog
GET /api/courses
GET /api/courses/{code}
```

Bind `{code}` to the method parameter.

The controller should delegate immediately to `CourseService`. Do not access the repository from the controller.

### 9.3 Verify the routes

```bash
curl -i http://localhost:8080/api/catalog
curl -i http://localhost:8080/api/courses
curl -i http://localhost:8080/api/courses/java-101
```

A possible catalogue response is:

```json
{
  "title": "Application Development Catalogue",
  "currentCourses": 2,
  "maximumCourses": 10
}
```

The course list must be a JSON array sorted by code.

### Configuration checkpoint

Restart with the `CATALOG_TITLE` environment variable and call `/api/catalog` again. The response should change without source-code modification or recompilation.

---

## 10. Task 4 - Create, validate, and translate failures 

### 10.1 Map the creation operation

In `CourseController`, map the existing `create(...)` method to:

```text
POST /api/courses
```

Bind the JSON body and activate validation. The method already constructs the `Location` header and returns `201 Created`.

Use this request:

```bash
curl -i \
  -X POST http://localhost:8080/api/courses \
  -H "Content-Type: application/json" \
  -d '{
    "code": "OPS-201",
    "title": "Application Operations",
    "credits": 6,
    "level": "INTERMEDIATE"
  }'
```

Expected response characteristics:

```text
HTTP/1.1 201
Location: /api/courses/OPS-201
```

The body must contain the created course.

### 10.2 Activate the global exception handler

Open:

```text
src/main/java/edu/catalog/api/error/GlobalExceptionHandler.java
```

Register it as REST advice, then map:

| Exception | Status |
|---|---:|
| `CourseNotFoundException` | `404 Not Found` |
| `DuplicateCourseException` | `409 Conflict` |
| `CatalogCapacityException` | `409 Conflict` |
| `MethodArgumentNotValidException` | `400 Bad Request` |

The response body is already represented by `ApiError`.

### 10.3 Verify one failure of each type

Unknown course:

```bash
curl -i http://localhost:8080/api/courses/NOPE-404
```

Duplicate course: repeat the successful POST request.

Invalid request:

```bash
curl -i \
  -X POST http://localhost:8080/api/courses \
  -H "Content-Type: application/json" \
  -d '{
    "code": "bad",
    "title": "x",
    "credits": 0,
    "level": null
  }'
```

A failure response should have a stable JSON structure similar to:

```json
{
  "timestamp": "2026-01-01T12:00:00Z",
  "status": 400,
  "error": "Bad Request",
  "message": "code: must follow CODE-123; title: size must be between 4 and 80",
  "path": "/api/courses"
}
```

The timestamp and exact validation message can differ.

---

## 11. Task 5 - Health, packaging, and runtime override 

### 11.1 Verify the exposed management endpoint

```bash
curl -i http://localhost:8080/actuator/health
```

Only `health` and `info` should be exposed over HTTP by the provided configuration.

### 11.2 Package the application

Stop the running application, then execute:

```bash
./mvnw clean package
```

On Windows:

```powershell
mvnw.cmd clean package
```

### 11.3 Run the executable JAR on another port

```bash
java -jar target/course-catalog-0.0.1-SNAPSHOT.jar --server.port=9090
```

Verify:

```bash
curl -i http://localhost:9090/actuator/health
```

This demonstrates that packaging and runtime configuration are separate concerns.

---

## 12. Acceptance checklist

### Startup and configuration

- [ ] The project starts with the provided launcher.
- [ ] The application uses the configured name and port.
- [ ] `CatalogProperties` receives the catalogue settings.
- [ ] `CATALOG_TITLE` overrides the file-based title.

### Managed object graph

- [ ] Repository and service are Spring-managed components.
- [ ] The controller receives its service through the constructor.
- [ ] The service receives the repository and typed properties through its constructor.
- [ ] No component performs a global bean lookup.
- [ ] No singleton component stores request-specific mutable state.

### HTTP behavior

- [ ] `GET /api/catalog` returns the title, current size, and maximum size.
- [ ] `GET /api/courses` returns the seeded and newly created courses.
- [ ] `GET /api/courses/{code}` normalizes the code.
- [ ] Missing courses return `404`.
- [ ] Valid creation returns `201` and a `Location` header.
- [ ] Invalid input returns `400`.
- [ ] Duplicate and capacity failures return `409`.

### Operations

- [ ] `/actuator/health` reports `UP`.
- [ ] The application packages as an executable JAR.
- [ ] A command-line argument can change the server port.

---

## 13. Troubleshooting guide

### The application reports that no bean is available

Read the dependency chain in the startup failure. Check whether every application component in that chain is registered and located below the root package scanned by `CatalogApplication`.

### The application reports multiple beans of the same type

Do not add a second implementation for this laboratory. Remove the accidental duplicate rather than hiding the ambiguity with `@Primary`.

### A route returns `404`, but the application is running

A healthy server does not guarantee that a controller mapping exists. Check:

- whether the class is a REST controller;
- the class-level route prefix;
- the method-level mapping;
- the HTTP method;
- the requested path.

### The POST request returns `415 Unsupported Media Type`

Include:

```text
Content-Type: application/json
```

### Validation does not run

Check that:

- the validation starter is present;
- constraints exist on `CreateCourseRequest`;
- the controller activates validation for the request body.

### Port 8080 is already in use

Run with a temporary override:

```bash
./mvnw spring-boot:run -Dspring-boot.run.arguments=--server.port=9090
```

---

## 14. Submission

Submit:

1. the completed repository;
2. a `README.md` containing the run and package commands;
3. one successful GET response;
4. one successful POST response showing `201` and `Location`;
5. one error response showing `400`, `404`, or `409`;
6. a short reflection of five to seven sentences.

Use a meaningful commit message, for example:

```text
Complete Spring Boot catalogue API laboratory
```

---

## 15. Reflection questions

Address these questions in the reflection:

1. Who creates `CourseController`, `CourseService`, and `InMemoryCourseRepository`?
2. What does constructor injection make visible?
3. Which configuration value changed without rebuilding the JAR?
4. What responsibilities belong to the controller, service, repository, and global advice?
5. Which parts of the application were supplied by auto-configuration?
6. What would need to change when the in-memory repository is replaced by a database adapter?

