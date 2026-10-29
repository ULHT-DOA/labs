# Module 03 Laboratory - Database Modelling, JPA, and Hibernate

## Persistent Course Catalogue
---

## 1. Purpose

In this laboratory, you will replace the incomplete persistence mapping of the **Course Catalogue API** with a working relational model and JPA implementation.

The starter repository already contains:

- a Spring Boot web application;
- a Flyway migration that creates and seeds the database;
- `Department` and `Course` entity classes with deliberate mapping gaps;
- Spring Data repository interfaces;
- request and response DTOs;
- a controller and service with focused `TODO` markers;
- an incomplete Mermaid ER diagram;
- HTTP requests for verification.

The objective is not to create a large database. The objective is to preserve the same business meaning across five representations:

```text
Business rule -> ER model -> relational schema -> JPA mapping -> observed SQL
```

You will work with one relationship:

> A department may offer zero or many courses. Every course belongs to exactly one department.

This laboratory intentionally does **not** include security, Docker, PostgreSQL, automated tests, or enrolment modelling. Those concerns would exceed the one-hour timebox.

---

## 2. Learning outcomes

After completing the laboratory, you should be able to:

1. Read cardinality and optionality from a written business rule.
2. Identify the table that owns a foreign key.
3. Explain how primary keys, unique constraints, check constraints, foreign keys, and indexes protect data integrity.
4. Map a required many-to-one association with `@ManyToOne` and `@JoinColumn`.
5. Map the inverse one-to-many side with `mappedBy`.
6. Explain why the many-to-one side owns the relationship in JPA.
7. derive a Spring Data query from a mapped object path.
8. implement a transactional create operation that loads a managed `Department` and persists a `Course`.
9. distinguish Flyway schema ownership from Hibernate schema validation.
10. relate repository operations and object navigation to the SQL shown in the application log.

---

## 3. Domain rules

The application uses the following rules:

### Department

- A department has an internal database identifier.
- A department code is required and unique.
- A department name is required.
- A department may exist without courses.

### Course

- A course has an internal database identifier.
- A course code is required and unique.
- A title is required.
- Credits must be between `1` and `30`.
- A level is stored using a stable textual value.
- Every course must reference exactly one existing department.

### Relationship

```text
DEPARTMENT 1 -------- 0..* COURSE
```

The relationship means:

- one department can be referenced by many courses;
- a department can currently have no courses;
- one course cannot exist without a department;
- the foreign key belongs in the `course` table because each course stores one department identifier.

---

## 4. Completed relational target

The intended schema is:

```text
DEPARTMENT
-----------
id              PK
code            NOT NULL, UNIQUE
name            NOT NULL

COURSE
------
id              PK
code            NOT NULL, UNIQUE
title           NOT NULL
credits         NOT NULL, CHECK 1..30
level           NOT NULL
department_id   NOT NULL, FK -> department.id
```

The starter migration already implements this schema. Your task is to make the ER diagram, Java mappings, repositories, and service behaviour agree with it.

---

## 5. Time plan

Respect the timebox. Stop when the acceptance criteria are satisfied.

| Activity | Time |
|---|---:|
| Start the application, inspect the migration, and complete the ER relationship | 8 minutes |
| Complete the Department-Course JPA relationship | 15 minutes |
| Complete the repository query and filtered read use case | 10 minutes |
| Implement the transactional create use case | 15 minutes |
| Enable validation, exercise the API, and inspect SQL | 12 minutes |
| **Total** | **60 minutes** |

---

## 6. Starter repository

```text
course-catalog-persistence/
├── README.md
├── pom.xml
├── mvnw
├── mvnw.cmd
├── requests.http
├── docs/
│   └── model.md
└── src/
    └── main/
        ├── java/edu/catalog/api/
        │   ├── CatalogApplication.java
        │   ├── course/
        │   │   ├── Course.java
        │   │   ├── CourseController.java
        │   │   ├── CourseLevel.java
        │   │   ├── CourseRepository.java
        │   │   ├── CourseResponse.java
        │   │   ├── CourseService.java
        │   │   └── CreateCourseRequest.java
        │   ├── department/
        │   │   ├── Department.java
        │   │   └── DepartmentRepository.java
        │   └── error/
        │       ├── ApiError.java
        │       ├── DepartmentNotFoundException.java
        │       ├── DuplicateCourseException.java
        │       └── GlobalExceptionHandler.java
        └── resources/
            ├── application.properties
            └── db/migration/
                └── V1__create_catalog_schema.sql
```

Search the project for `TODO`. Every marker belongs to a task in this document.

---

## 7. Task 0 - Start and inspect the persistence layer (8 minutes)

### 7.1 Run the application

On macOS or Linux:

```bash
./mvnw spring-boot:run
```

On Windows PowerShell:

```powershell
.\mvnw.cmd spring-boot:run
```

The first execution can take longer because Maven downloads dependencies.

### 7.2 Verify the baseline

In another terminal:

```bash
curl -s http://localhost:8080/api/courses
```

The starter can read the three seeded course rows, but each response initially shows:

```json
"departmentCode": null
```

This is deliberate. The database contains `department_id`, but the Java association is temporarily marked `@Transient`. The mismatch makes the mapping problem visible.

The department-filtered route currently returns an empty array:

```bash
curl -s "http://localhost:8080/api/courses?department=COMP"
```

The POST route currently returns `501 Not Implemented`:

```bash
curl -i -X POST http://localhost:8080/api/courses \
  -H "Content-Type: application/json" \
  -d '{
    "code": "ARCH-301",
    "title": "Software Architecture",
    "credits": 6,
    "level": "ADVANCED",
    "departmentCode": "COMP"
  }'
```

### 7.3 Inspect the migration

Open:

```text
src/main/resources/db/migration/V1__create_catalog_schema.sql
```

Find evidence for each rule:

- primary keys;
- unique department and course codes;
- non-null columns;
- the credits check constraint;
- the `course.department_id` foreign key;
- the index on `course.department_id`;
- seed inserts.

Do not edit the migration during the mandatory laboratory.

### 7.4 Complete the ER relationship

Open:

```text
docs/model.md
```

Add the missing Mermaid relationship:

```mermaid
DEPARTMENT ||--o{ COURSE : offers
```

Interpret the symbols in a short sentence below the diagram:

- `||` at `DEPARTMENT`: each course references exactly one department;
- `o{` at `COURSE`: a department may offer zero or many courses.

### Checkpoint

Before continuing, identify the relationship owner in the relational schema. The answer is the table that stores the foreign key.

---

## 8. Task 1 - Complete the JPA relationship (15 minutes)

Open:

```text
src/main/java/edu/catalog/api/course/Course.java
src/main/java/edu/catalog/api/department/Department.java
```

### 8.1 Map the owning side

In `Course`, replace the temporary `@Transient` mapping on `department`.

The completed mapping must express:

- many courses can reference one department;
- a course cannot omit its department;
- loading a course must not automatically require eager loading of its department;
- the association uses the existing `department_id` column;
- the foreign-key name agrees with the migration.

Use these JPA elements:

```java
@ManyToOne(fetch = FetchType.LAZY, optional = false)
@JoinColumn(
    name = "department_id",
    nullable = false,
    foreignKey = @ForeignKey(name = "fk_course_department")
)
```

### 8.2 Map the inverse side

In `Department`, replace the temporary `@Transient` mapping on `courses`.

Use a one-to-many association whose `mappedBy` value names the Java field on `Course`:

```java
@OneToMany(mappedBy = "department")
```

Do **not** add a second join column or a join table. The relational schema already has one source of truth: `course.department_id`.

### 8.3 Preserve both sides in memory

Inspect:

```java
Department.addCourse(...)
Course.assignDepartment(...)
```

The helper must keep both Java references synchronized:

- the department collection contains the course;
- the course references the department.

This synchronization does not move relationship ownership. Hibernate still persists the relationship from `Course.department`, because that field maps the foreign key.

### 8.4 Restart and verify

Restart the application and call:

```bash
curl -s http://localhost:8080/api/courses
```

The seeded rows should now include department codes, for example:

```json
{
  "id": 1,
  "code": "JAVA-101",
  "title": "Java Programming Foundations",
  "credits": 6,
  "level": "FOUNDATIONAL",
  "departmentCode": "COMP"
}
```

### Checkpoint

Explain why this mapping is not simply "two annotations for the same thing":

- `Course.department` is the owning side and writes the foreign key;
- `Department.courses` is the inverse navigation side and uses `mappedBy`.

---

## 9. Task 2 - Complete the repository query (10 minutes)

Open:

```text
src/main/java/edu/catalog/api/course/CourseRepository.java
src/main/java/edu/catalog/api/course/CourseService.java
```

### 9.1 Add a derived query

Add this repository method:

```java
List<Course> findByDepartmentCodeOrderByCode(String departmentCode);
```

Read the method from left to right:

```text
find Course
where Course.department.code equals the argument
order by Course.code
```

The method depends on the object mapping created in Task 1. Spring Data derives JPQL from entity property paths; it does not derive this query from column names.

### 9.2 Complete the filtered use case

Complete `CourseService.findByDepartment(...)` so that it:

1. normalizes the incoming department code with `trim()` and `toUpperCase(Locale.ROOT)`;
2. calls the new repository method;
3. maps every entity to `CourseResponse`;
4. returns the resulting list.

### 9.3 Verify

```bash
curl -s "http://localhost:8080/api/courses?department=COMP"
```

Expected course codes:

```text
JAVA-101
WEB-201
```

The `DATA` department should return:

```bash
curl -s "http://localhost:8080/api/courses?department=DATA"
```

Expected course code:

```text
DB-210
```

### Checkpoint

Locate the generated SQL in the log. Identify:

- the course table alias;
- the join to the department table, if present;
- the predicate derived from `department.code`;
- the `order by` generated from the method name.

---

## 10. Task 3 - Implement the transactional create use case (15 minutes)

Open:

```text
src/main/java/edu/catalog/api/course/CourseService.java
```

Complete `create(...)` and mark the method with:

```java
@Transactional
```

The operation must perform these steps within one transaction:

1. normalize the course code and department code;
2. reject an existing course code using `existsByCodeIgnoreCase(...)`;
3. load the department with `findByCodeIgnoreCase(...)`;
4. throw `DepartmentNotFoundException` when the department does not exist;
5. construct a `Course` from the validated request;
6. synchronize the relationship with `department.addCourse(course)`;
7. save the owning entity with `courseRepository.save(course)`;
8. map the saved entity to `CourseResponse`.

Do not create a new `Department` from the incoming code. The course must reference the managed department row already stored in the database.

### Why the transaction matters

The transaction defines one persistence unit of work:

```text
validate request
  -> load Department
  -> create Course
  -> establish association
  -> INSERT Course
  -> commit
```

If an exception occurs before commit, the operation must not leave a partially completed database change.

### Verify successful creation

```bash
curl -i -X POST http://localhost:8080/api/courses \
  -H "Content-Type: application/json" \
  -d '{
    "code": "ARCH-301",
    "title": "Software Architecture",
    "credits": 6,
    "level": "ADVANCED",
    "departmentCode": "COMP"
  }'
```

Expected status:

```text
201 Created
```

Expected response properties:

```json
{
  "code": "ARCH-301",
  "title": "Software Architecture",
  "credits": 6,
  "level": "ADVANCED",
  "departmentCode": "COMP"
}
```

The generated identifier can vary.

### Verify filtered reading after the write

```bash
curl -s "http://localhost:8080/api/courses?department=COMP"
```

`ARCH-301` should now appear with the two seeded COMP courses.

---

## 11. Task 4 - Validate mapping, constraints, and SQL (12 minutes)

### 11.1 Enable Hibernate schema validation

Open:

```text
src/main/resources/application.properties
```

Change:

```properties
spring.jpa.hibernate.ddl-auto=none
```

To:

```properties
spring.jpa.hibernate.ddl-auto=validate
```

Restart the application.

The responsibilities are now explicit:

- Flyway applies the versioned schema migration;
- Hibernate validates that entity mappings are compatible with that schema;
- Hibernate does not create or update the schema.

### 11.2 Exercise validation and error translation

#### Invalid credits

```bash
curl -i -X POST http://localhost:8080/api/courses \
  -H "Content-Type: application/json" \
  -d '{
    "code": "BAD-001",
    "title": "Invalid Course",
    "credits": 0,
    "level": "FOUNDATIONAL",
    "departmentCode": "COMP"
  }'
```

Expected status:

```text
400 Bad Request
```

#### Unknown department

```bash
curl -i -X POST http://localhost:8080/api/courses \
  -H "Content-Type: application/json" \
  -d '{
    "code": "CLOUD-310",
    "title": "Cloud Platforms",
    "credits": 6,
    "level": "ADVANCED",
    "departmentCode": "UNKNOWN"
  }'
```

Expected status:

```text
404 Not Found
```

#### Duplicate course code

Repeat the valid `ARCH-301` request.

Expected status:

```text
409 Conflict
```

The service check produces a useful API response. The database unique constraint remains the final integrity guard against races or changes made outside the application.

### 11.3 Inspect the SQL

Use the application log to identify SQL corresponding to:

- listing courses;
- resolving a department by code;
- checking whether a course code exists;
- inserting the new course;
- filtering courses through `department.code`.

For one request, write a short trace in this form:

```text
HTTP request
  -> controller method
  -> service method
  -> repository method
  -> generated SQL
  -> response DTO
```

### Optional local inspection

The H2 console is enabled at:

```text
http://localhost:8080/h2-console
```

Use:

```text
JDBC URL: jdbc:h2:mem:catalog;MODE=PostgreSQL;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
User: sa
Password: <empty>
```

A browser-based console is optional; the API and log are sufficient for the laboratory.

---

## 12. Acceptance criteria

The laboratory is complete when all mandatory statements are true.

### Model and schema

- [ ] `docs/model.md` shows `DEPARTMENT ||--o{ COURSE : offers`.
- [ ] The relationship explanation identifies cardinality and optionality correctly.
- [ ] You can identify `course.department_id` as the foreign key and relational owner.
- [ ] You can point to the primary, unique, check, foreign-key, and index definitions in the migration.

### JPA and Hibernate

- [ ] `Course.department` uses required lazy `@ManyToOne` mapping.
- [ ] `Course.department` maps `department_id`.
- [ ] `Department.courses` uses `@OneToMany(mappedBy = "department")`.
- [ ] No duplicate join column or join table was introduced.
- [ ] Both Java sides are synchronized when a course is added.
- [ ] `ddl-auto=validate` starts successfully against the Flyway-managed schema.

### Repository and transaction

- [ ] The derived repository query filters through `Course.department.code`.
- [ ] The COMP filter returns the expected courses in code order.
- [ ] The create use case runs inside one transaction.
- [ ] A new course references an existing managed department.
- [ ] A valid POST returns `201 Created`.
- [ ] Invalid input returns `400`.
- [ ] An unknown department returns `404`.
- [ ] A duplicate code returns `409`.

### Runtime understanding

- [ ] You can connect at least one repository operation to its generated SQL.
- [ ] You can explain the different responsibilities of Flyway and Hibernate validation.

---

## 13. Submission

Commit the following:

```text
docs/model.md
src/main/java/edu/catalog/api/course/Course.java
src/main/java/edu/catalog/api/course/CourseRepository.java
src/main/java/edu/catalog/api/course/CourseService.java
src/main/java/edu/catalog/api/department/Department.java
src/main/resources/application.properties
```

Use a commit message such as:

```text
Complete department-course persistence mapping
```

Do not commit generated build output, the H2 database, IDE folders, or downloaded dependencies.

---

## 14. Reflection questions

Answer briefly in your submission or be prepared to discuss them.

1. Why does `Course` own the JPA relationship even though `Department` has a collection of courses?
2. What incorrect schema could result from placing `@JoinColumn` on both sides?
3. Which rule is protected by `optional = false`, `nullable = false`, and the database `NOT NULL` constraint? Why is the rule expressed at more than one boundary?
4. What would change if the business rule allowed a course without a department?
5. Why is a unique service check not a substitute for a database unique constraint?
6. What does `mappedBy = "department"` refer to: a Java field or a database column?
7. Why does `findByDepartmentCodeOrderByCode` require a valid entity association?
8. What is the difference between flushing SQL and committing a transaction?
9. Why is `ddl-auto=update` unsuitable as the only production schema-evolution strategy?
10. Where could an N+1 query problem appear when course responses access department data?

---

## 15. Optional extensions - only after the mandatory work

These extensions are deliberately outside the 60-minute requirement.

### Extension A - Explicit fetch plan

Add a repository query or entity graph that loads courses with departments for the list endpoint. Compare the SQL before and after the change.

### Extension B - Optimistic locking

Add a `@Version` field to `Course`, create a matching Flyway migration, and explain which concurrent update problem it detects.

### Extension C - Enrolment associative entity

Model `Student` and `Enrolment` so that enrolment carries:

- enrolment date;
- status;
- final grade;
- a unique constraint over `(student_id, course_id)`.

Do not replace `Enrolment` with a direct many-to-many mapping, because the relationship contains its own data and identity.

### Extension D - PostgreSQL profile

Add a PostgreSQL runtime profile and a new development database configuration. Keep the Flyway migrations as the source of schema evolution.

---

## 16. Scope boundary

During the mandatory hour, do not add:

- security;
- Docker or Docker Compose;
- PostgreSQL;
- automated tests;
- Lombok;
- bidirectional JSON serialization of entities;
- a generic repository abstraction;
- an enrolment model;
- a many-to-many shortcut;
- cascading deletes;
- a second schema migration.

A small, correct model is more valuable than a large unfinished one.
