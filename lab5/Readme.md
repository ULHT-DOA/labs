# Laboratory 05 - Securing the Workshop Registration API


This laboratory adds a deliberately small security boundary to an existing Spring Boot API. The application already implements workshop discovery, registration, and organizer operations. Your task is to decide which requests are public, which require an authenticated identity, and which require organizer authority.

The objective is not to create a complete identity platform. It is to understand the core Spring Security object graph and to make a few explicit application-security decisions.

---

## Learning outcomes

By the end of the laboratory, you should be able to:

- Add users whose passwords are stored through a `PasswordEncoder`.
- Explain the difference between authentication and authorization.
- Configure a `SecurityFilterChain` with least-privilege request rules.
- Distinguish `401 Unauthorized` from `403 Forbidden`.
- Read the authenticated identity from Spring Security instead of trusting identity fields supplied by a client.
- Enable method security and protect a service operation with `@PreAuthorize`.
- Explain why the laboratory's CSRF decision depends on its client assumptions.

---

## Scenario

The Workshop Registration API has three kinds of operations:

1. **Public discovery** - anyone may view the available workshops.
2. **Participant self-service** - an authenticated participant may register themselves.
3. **Organizer administration** - only organizers may create workshops and inspect participant lists.

The starter configuration is intentionally insecure:

```java
.authorizeHttpRequests(authorize -> authorize
        .anyRequest().permitAll()
)
```

Do not add security rules blindly. First write the expected access matrix.

---

## Accounts used in the laboratory

| Username | Password | Role |
|---|---|---|
| `participant@example.com` | `participant123` | `PARTICIPANT` |
| `organizer@example.com` | `organizer123` | `ORGANIZER` |

These accounts are local teaching data. Production passwords must not be embedded in source code.

---

## Expected access matrix

Complete `docs/security-decisions.md` before changing the code.

| Operation | Anonymous | Participant | Organizer |
|---|---:|---:|---:|
| List and view workshops | Allowed | Allowed | Allowed |
| View the current identity | Rejected | Allowed | Allowed |
| Register for a workshop | Rejected | Allowed | Allowed |
| Create a workshop | Rejected | Rejected | Allowed |
| View workshop registrations | Rejected | Rejected | Allowed |

An unauthenticated request to a protected resource should receive **401**. An authenticated user without sufficient authority should receive **403**.

---

## Time plan

| Activity | Maximum time |
|---|---:|
| 1. Start the API and observe the insecure baseline | 5 minutes |
| 2. Add encoded users | 10 minutes |
| 3. Configure request-level authorization | 18 minutes |
| 4. Use the authenticated identity and enable method security | 12 minutes |
| 5. Verify 200, 201, 401, and 403 responses | 10 minutes |
| 6. Record the CSRF decision and review | 5 minutes |
| **Total** | **60 minutes** |

---

# 1. Start and inspect the insecure baseline

Run the application:

```bash
./mvnw spring-boot:run
```

Windows PowerShell:

```powershell
.\mvnw.cmd spring-boot:run
```

Confirm that the following request works without credentials:

```bash
curl -i http://localhost:8080/api/workshops
```

Now try the administrative endpoint without credentials:

```bash
curl -i http://localhost:8080/api/admin/workshops/22222222-2222-2222-2222-222222222222/registrations
```

It also succeeds because the starter rule permits every request. This is the risk you will remove.

### Checkpoint

Before continuing, locate:

- `SecurityConfiguration`
- `WorkshopController`
- `AdminWorkshopController`
- `ProfileController`
- `WorkshopService.listRegistrations(...)`
- `RestAuthenticationEntryPoint`
- `RestAccessDeniedHandler`

---

# 2. Add encoded users - 10 minutes

Open `SecurityConfiguration`.

## 2.1 Add a password encoder

Expose a `PasswordEncoder` created by:

```java
PasswordEncoderFactories.createDelegatingPasswordEncoder()
```

Do not use `NoOpPasswordEncoder`, plain-text passwords, or a fast general-purpose digest such as SHA-256 for password storage.

## 2.2 Add the two in-memory users

Expose an `InMemoryUserDetailsManager` containing:

- `participant@example.com` with role `PARTICIPANT`
- `organizer@example.com` with role `ORGANIZER`

Encode both passwords through the configured encoder.

A useful shape is:

```java
UserDetails participant = User.withUsername(/* ... */)
        .password(/* encoded password */)
        .roles(/* ... */)
        .build();
```

### Checkpoint

Restart the application. There should no longer be a generated development password in the startup log.

---

# 3. Configure request-level authorization 

Replace the permissive rule set in `securityFilterChain(...)`.

## 3.1 Authentication mechanism

Enable HTTP Basic authentication for this command-line laboratory:

```java
.httpBasic(Customizer.withDefaults())
```

Configure stateless request handling:

```java
.sessionManagement(session -> session
        .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
)
```

## 3.2 Request rules

Implement the access matrix with `authorizeHttpRequests`.

The following route groups are available:

| Route | Intended rule |
|---|---|
| `GET /api/workshops` | Public |
| `GET /api/workshops/{id}` | Public |
| `GET /api/me` | Authenticated |
| `POST /api/workshops/{id}/registrations` | `PARTICIPANT` or `ORGANIZER` |
| `/api/admin/**` | `ORGANIZER` |
| Any unlisted request | Denied |

Use `HttpMethod` matchers where the HTTP method is part of the policy. Finish with `anyRequest().denyAll()` rather than accidentally opening future endpoints.

**Ordering matters.** Spring Security evaluates request rules in declaration order and uses the first matching rule.

## 3.3 Explicit REST error handling

Connect the supplied handlers:

```java
.exceptionHandling(exceptions -> exceptions
        .authenticationEntryPoint(authenticationEntryPoint)
        .accessDeniedHandler(accessDeniedHandler)
)
```

The entry point should produce `401` and a `WWW-Authenticate` header. The access-denied handler should produce `403`.

### Checkpoint

Run these requests:

```bash
curl -i http://localhost:8080/api/workshops
curl -i http://localhost:8080/api/me
curl -i -u participant@example.com:participant123 http://localhost:8080/api/me
```

Expected status codes:

```text
GET /api/workshops                 -> 200
GET /api/me                        -> 401
GET /api/me as participant         -> 200
```

---

# 4. Use authenticated identity and method security

## 4.1 Do not trust identity from the request body

Inspect `WorkshopController.register(...)`.

The request body contains only the participant's display name. The email used by the registration comes from:

```java
principal.getName()
```

This prevents a participant from claiming another person's identity by changing a JSON field.

Do not add `participantEmail` to `RegisterParticipantRequest`.

## 4.2 Enable method security

Add `@EnableMethodSecurity` to the security configuration.

Then protect `WorkshopService.listRegistrations(...)` with:

```java
@PreAuthorize("hasRole('ORGANIZER')")
```

This creates a second authorization boundary around the sensitive service operation. The request rule remains useful, but the service no longer relies exclusively on a particular controller path.

### Checkpoint

A participant must not be able to invoke the registration-list operation even if a future controller is mapped incorrectly.

---

# 5. Verify the security boundary - 10 minutes

The repository contains `requests.http`. You may also use the commands below.

## 5.1 Public read

```bash
curl -i http://localhost:8080/api/workshops
```

Expected: `200 OK`.

## 5.2 Protected endpoint without credentials

```bash
curl -i http://localhost:8080/api/me
```

Expected: `401 Unauthorized` and a `WWW-Authenticate` response header.

## 5.3 Participant registration

```bash
curl -i \
  -u participant@example.com:participant123 \
  -H 'Content-Type: application/json' \
  -d '{"participantName":"Ana Martins"}' \
  http://localhost:8080/api/workshops/22222222-2222-2222-2222-222222222222/registrations
```

Expected: `201 Created`. The response email must be `participant@example.com`, derived from the authenticated identity.

## 5.4 Participant attempts an organizer operation

```bash
curl -i \
  -u participant@example.com:participant123 \
  -H 'Content-Type: application/json' \
  -d '{"title":"Secure API Design","description":"Threat modelling and secure defaults.","scheduledDate":"2027-05-06","capacity":18}' \
  http://localhost:8080/api/admin/workshops
```

Expected: `403 Forbidden`.

## 5.5 Organizer performs the operation

```bash
curl -i \
  -u organizer@example.com:organizer123 \
  -H 'Content-Type: application/json' \
  -d '{"title":"Secure API Design","description":"Threat modelling and secure defaults.","scheduledDate":"2027-05-06","capacity":18}' \
  http://localhost:8080/api/admin/workshops
```

Expected: `201 Created`.

## 5.6 Organizer views registrations

```bash
curl -i \
  -u organizer@example.com:organizer123 \
  http://localhost:8080/api/admin/workshops/22222222-2222-2222-2222-222222222222/registrations
```

Expected: `200 OK`.

---

# 6. Record the CSRF decision 

The starter disables CSRF so that the command-line API exercise can focus on the authentication and authorization flow.

This is **not** a universal REST rule. Complete the questions in `docs/security-decisions.md`:

1. Which clients are assumed by the laboratory?
2. Are cookies used?
3. Are credentials automatically attached by a browser?
4. Would the decision remain valid for a browser application using session cookies?
5. What protection would be required in that case?

Also record at least four values that should never be written to ordinary application logs, such as passwords or full authorization headers.


# Submission

Commit:

- The completed `SecurityConfiguration`.
- The method-security annotation on `WorkshopService`.
- The completed `docs/security-decisions.md`.
- A short `RESULTS.md` containing the observed status codes for the six verification requests.

Do not commit passwords other than the local laboratory credentials already specified in this exercise.

---

# Reflection questions

1. Why is a valid username and password not sufficient to access every endpoint?
2. Why does an unauthenticated request produce `401`, while an authenticated participant receives `403` for an organizer operation?
3. What risk would be introduced if the registration request accepted an arbitrary participant email?
4. What does request authorization protect that method authorization does not, and vice versa?
5. Why is password encoding different from encryption?
6. Why is disabling CSRF a decision based on client and credential behaviour rather than on the word “REST”?
