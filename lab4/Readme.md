# Laboratory - Workshop Registration Safety Net

## Purpose

In this laboratory you will create a small, deliberate test portfolio for an existing Spring Boot service.

The production code is already implemented. You are **not** expected to create controllers, persistence mappings, database schemas, or new application features. Your work is limited to the testing concepts introduced in Module 04:

- test scope and evidence;
- JUnit Jupiter;
- AssertJ;
- parameterized tests;
- Mockito test doubles;
- deterministic time with `Clock`;
- Spring TestContext architecture;
- `@SpringBootTest`, `@TestConfiguration`, and `@MockitoBean`.


Do not add optional work until the mandatory acceptance criteria are complete.

---

## Scenario

The application registers participants in technical workshops.

A registration is accepted only when:

1. the workshop exists;
2. the workshop is not in the past;
3. the participant email is not already registered in the same workshop;
4. the workshop still has an available place.

The service normalizes participant emails by trimming whitespace and converting them to lowercase. A successful registration records the current instant.

The application contains the following important collaborators:

```text
RegisterParticipantCommand
            |
            v
    RegistrationService
       /      |       \
      v       v        v
Workshop   Registration  Clock
Repository Repository
```

The repositories have in-memory production implementations. In the unit tests you will replace the interfaces with Mockito mocks. In the final task, Spring will construct the real service while `@MockitoBean` replaces the repository beans inside the application context.

---

## Repository structure

```text
workshop-registration-testing/
├── pom.xml
├── README.md
├── LABORATORY.md
└── src/
    ├── main/
    │   ├── java/edu/workshops/
    │   │   ├── WorkshopRegistrationApplication.java
    │   │   ├── config/
    │   │   │   └── TimeConfiguration.java
    │   │   └── registration/
    │   │       ├── Workshop.java
    │   │       ├── Registration.java
    │   │       ├── RegisterParticipantCommand.java
    │   │       ├── RegistrationService.java
    │   │       ├── WorkshopRepository.java
    │   │       ├── RegistrationRepository.java
    │   │       ├── InMemoryWorkshopRepository.java
    │   │       ├── InMemoryRegistrationRepository.java
    │   │       └── ...exceptions
    │   └── resources/
    │       └── application.properties
    └── test/
        └── java/edu/workshops/
            ├── WorkshopRegistrationApplicationTest.java
            └── registration/
                ├── WorkshopTest.java
                ├── RegistrationServiceTest.java
                └── RegistrationServiceContextTest.java
```

---

## 0. Run and inspect the baseline 

From the project root, run:

```bash
./mvnw test
```

On Windows:

```powershell
.\mvnw.cmd test
```

The starter should pass because:

- `WorkshopRegistrationApplicationTest` is enabled;
- the three exercise classes are initially marked with `@Disabled`;
- the application context can be constructed using the production in-memory repositories and system `Clock`.

Before changing code, inspect:

- `Workshop`;
- `RegistrationService`;
- `WorkshopRepository`;
- `RegistrationRepository`;
- the three disabled exercise classes.

### Discussion checkpoint

Be ready to explain:

1. Which rules belong directly to `Workshop`?
2. Which rules require repository information?
3. Why is `Clock` a constructor dependency?
4. Which tests can run without Spring?

---

## 1. Test the `Workshop` rules 

Open:

```text
src/test/java/edu/workshops/registration/WorkshopTest.java
```

Remove the class-level `@Disabled` annotation.

### 1.1 Invalid capacity

Complete `rejectsZeroCapacity()`.

The test must:

- attempt to construct a workshop with capacity `0`;
- use `assertThatThrownBy`;
- assert `IllegalArgumentException`;
- assert that the message mentions `capacity`.

Do not start Spring and do not use Mockito.

### 1.2 Capacity boundary

Complete the parameterized test `evaluatesAvailablePlaces(...)`.

The supplied cases are:

| Accepted registrations | Capacity | Expected availability |
|---:|---:|---|
| 19 | 20 | `true` |
| 20 | 20 | `false` |

The equality case is the important boundary: a workshop is full when the accepted count reaches capacity.

Add one additional case of your own.

### 1.3 Date boundary

Complete:

- `workshopScheduledBeforeTodayIsPast()`;
- `workshopScheduledTodayIsNotPast()`.

Use explicit `LocalDate` values. Do **not** call `LocalDate.now()` in these tests.

### Checkpoint

Run only this class:

```bash
./mvnw -Dtest=WorkshopTest test
```

You should be able to explain why these are unit tests and why a Spring context would add no useful evidence.

---

## 2. Unit-test `RegistrationService` 

Open:

```text
src/test/java/edu/workshops/registration/RegistrationServiceTest.java
```

Remove the class-level `@Disabled` annotation.

This test class already provides:

- `@ExtendWith(MockitoExtension.class)`;
- mocked repository interfaces;
- a fixed `Clock`;
- direct construction of `RegistrationService`.

No Spring context is loaded.

### 2.1 Successful registration

Complete `registersParticipantWhenWorkshopHasCapacity()`.

Arrange the following scenario:

- the workshop exists;
- its date is `2027-04-15`;
- its capacity is `20`;
- no duplicate email exists;
- the repository reports `3` accepted registrations;
- `save(...)` returns the registration passed to it.

Use a command whose email contains uppercase characters and surrounding spaces, for example:

```text
" ANA@EXAMPLE.COM "
```

Assert that:

- the result email is `ana@example.com`;
- the result workshop identifier is correct;
- `registeredAt` equals the fixed instant `2027-04-01T09:00:00Z`;
- the registration repository receives a `save(...)` call.

Use an `ArgumentCaptor<Registration>` to inspect the object sent to `save(...)`.

### 2.2 Unknown workshop

Complete `rejectsUnknownWorkshop()`.

Arrange `workshopRepository.findById(...)` to return `Optional.empty()`.

Assert that:

- `WorkshopNotFoundException` is thrown;
- the exception message contains the workshop identifier;
- the registration repository is never asked to save.

### 2.3 Duplicate email

Complete `rejectsDuplicateEmailInSameWorkshop()`.

Arrange:

- an existing future workshop;
- `existsByWorkshopIdAndEmail(...)` returning `true`.

Assert that:

- `DuplicateRegistrationException` is thrown;
- the normalized email appears in the message;
- `countByWorkshopId(...)` is not called;
- `save(...)` is not called.

### 2.4 Full workshop

Complete `rejectsFullWorkshop()`.

Arrange:

- a workshop with capacity `20`;
- no duplicate registration;
- a repository count of `20`.

Assert that:

- `WorkshopFullException` is thrown;
- `save(...)` is not called.

### Checkpoint

Run only this class:

```bash
./mvnw -Dtest=RegistrationServiceTest test
```

Be prepared to state what these tests prove and what they cannot prove.

They can prove service decisions against controlled collaborator answers. They cannot prove component scanning, bean selection, configuration profiles, or Spring dependency injection.

---

## 3. Complete deterministic time cases

Continue in `RegistrationServiceTest`.

The test fixture already uses:

```java
private static final Instant NOW =
        Instant.parse("2027-04-01T09:00:00Z");

Clock fixedClock = Clock.fixed(NOW, ZoneOffset.UTC);
```

### 3.1 Past workshop

Complete `rejectsWorkshopScheduledBeforeTheFixedDate()`.

Use a workshop scheduled for `2027-03-31`.

Assert that:

- `WorkshopClosedException` is thrown;
- the duplicate check is not executed;
- no registration is saved.

### 3.2 Workshop scheduled today

Complete `acceptsWorkshopScheduledOnTheFixedDate()`.

Use a workshop scheduled for `2027-04-01`.

Complete a successful scenario and assert that:

- the operation is accepted;
- `registeredAt` equals `NOW`.

### Reflection checkpoint

Explain why a test based on `LocalDate.now()` or `Instant.now()` without an injected `Clock` could fail in the future even if the code has not changed.

---

## 4. Verify the Spring object graph 

Open:

```text
src/test/java/edu/workshops/registration/RegistrationServiceContextTest.java
```

Remove the class-level `@Disabled` annotation.

This is the only mandatory exercise test that loads Spring.

The class already contains:

```java
@SpringBootTest(webEnvironment = NONE)
@Import(FixedClockConfiguration.class)
```

It also declares:

```java
@MockitoBean
WorkshopRepository workshopRepository;

@MockitoBean
RegistrationRepository registrationRepository;

@Autowired
RegistrationService service;
```

The nested `@TestConfiguration` contributes a fixed, primary `Clock` bean.

### Complete the context scenario

Complete `serviceUsesTheBeansProvidedByTheTestContext()`.

Arrange a successful registration using the two `@MockitoBean` repositories.

Assert that:

- the autowired service is not null;
- the result email is normalized;
- `registeredAt` equals `NOW`;
- the registration is saved through the repository mock.

### What this test adds

Compared with `RegistrationServiceTest`, this test additionally proves that:

- Spring discovers `RegistrationService`;
- constructor injection succeeds;
- the repository beans in the context can be replaced;
- the imported fixed `Clock` is selected;
- the service receives the context-managed dependencies.

It still does not prove:

- HTTP request binding;
- controller behaviour;
- JPA mappings;
- database constraints;
- transaction behaviour;
- operation through a running server.

---

## Final verification

Run the complete suite:

```bash
./mvnw test
```

The final suite should contain:

- the provided context smoke test;
- domain boundary tests;
- service unit tests;
- one deliberate context-wiring test.

All tests must be deterministic and repeatable.

---

## Submission

Submit:

1. the completed repository;
2. the Maven test summary or a screenshot showing the passing suite;
3. a short Markdown note answering the reflection questions below.

## Reflection questions

1. Which tests required no Spring context, and why?
2. Why is `@Mock` appropriate in `RegistrationServiceTest`?
3. Why is `@MockitoBean` appropriate in `RegistrationServiceContextTest`?
4. What new evidence does the context test provide?
5. What would be wasteful about repeating every unit scenario through `@SpringBootTest`?
6. Which future risk would justify adding an MVC test?

