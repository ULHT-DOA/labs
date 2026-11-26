# Laboratory 06 - Applying the API Gateway Pattern

## QuickCart: one public entry point for two services

**Required Java version:** 17 or newer  
**Main technology:** Spring Cloud Gateway Server WebFlux

---

## 1. Purpose

QuickCart currently exposes two small Spring Boot services directly:

- **Catalogue Service** on port `8081`
- **Order Service** on port `8082`

A client must therefore know which service owns each operation and where that service is running. Your task is to introduce an **API Gateway** on port `8080` and give the client one stable public API.

The laboratory applies only one microservices pattern. It deliberately does **not** introduce service discovery, authentication, circuit breakers, messaging, or databases.

---

## 2. Starting architecture

```text
Client --------------------> Catalogue Service :8081
   |
   +-----------------------> Order Service     :8082
```

The client knows internal ports and internal paths. That creates unnecessary coupling between the client and the service topology.

## 3. Target architecture

```text
                         +--> Catalogue Service :8081  /products
Client --> Gateway :8080 |
                         +--> Order Service     :8082  /orders
```

The client should use only these public routes:

| Client operation | Public gateway route | Downstream route |
|---|---|---|
| List products | `GET /api/catalog/products` | `GET /products` |
| Find a product | `GET /api/catalog/products/{code}` | `GET /products/{code}` |
| List orders | `GET /api/orders` | `GET /orders` |
| Create an order | `POST /api/orders` | `POST /orders` |
| Find an order | `GET /api/orders/{id}` | `GET /orders/{id}` |

The client must not need to know ports `8081` or `8082` after the gateway is configured.

---

## 4. Learning outcomes

By the end of the laboratory, you should be able to:

1. Explain the client-side problem solved by an API Gateway.
2. Define a Spring Cloud Gateway route using a target URI and a path predicate.
3. Transform a public path before forwarding it to a downstream service.
4. Apply a default filter and a route-specific filter.
5. Verify gateway routing using HTTP requests and response headers.
6. Explain what belongs at the edge and what must remain inside a business service.
7. Observe what the gateway returns when a downstream service is unavailable.

---

## 5. Repository structure

```text
quickcart-api-gateway-lab/
├── api-gateway/
│   └── src/main/resources/application.yml
├── catalog-service/
├── order-service/
├── docs/
│   └── architecture-decisions.md
├── requests.http
├── pom.xml
├── mvnw
└── mvnw.cmd
```

The Catalogue and Order services are already implemented. Most of your work takes place in:

```text
api-gateway/src/main/resources/application.yml
```

---

# Task 0 - Start and inspect the applications

## 0.1 Build the project

From the repository root:

### macOS or Linux

```bash
./mvnw clean package
```

### Windows

```powershell
mvnw.cmd clean package
```

## 0.2 Start each application in a separate terminal

### Terminal 1 - Catalogue

```bash
./mvnw -pl catalog-service spring-boot:run
```

### Terminal 2 - Order

```bash
./mvnw -pl order-service spring-boot:run
```

### Terminal 3 - Gateway

```bash
./mvnw -pl api-gateway spring-boot:run
```

Windows users can replace `./mvnw` with `mvnw.cmd`.

## 0.3 Verify the direct service endpoints

```bash
curl -i http://localhost:8081/products
curl -i http://localhost:8082/orders
curl -i http://localhost:8080/actuator/health
```

At this point, the gateway health endpoint should work, but the business routes should not yet be available.

### Checkpoint

You should be able to answer:

- Which application owns `/products`?
- Which application owns `/orders`?
- Why should an external client not need to know those ports?

---

# Task 1 - Configure the Catalogue route

Open:

```text
api-gateway/src/main/resources/application.yml
```

Add one route with the following responsibilities:

| Route property | Required value |
|---|---|
| Route identifier | `catalog-service` |
| Target URI | `http://localhost:8081` |
| Public path | `/api/catalog/**` |
| Prefix transformation | Remove the first two path segments |

The public request:

```text
/api/catalog/products
```

must become:

```text
/products
```

before it reaches the Catalogue service.

Use:

- a **Path predicate** to select the route;
- a **StripPrefix filter** to transform the path.

Restart only the gateway after changing the YAML file.

## Verify

```bash
curl -i http://localhost:8080/api/catalog/products
curl -i http://localhost:8080/api/catalog/products/SKU-77
```

### Checkpoint

The client now reaches Catalogue through port `8080`. Catalogue still owns product behaviour; the gateway only selects and adapts the route.

---

# Task 2 - Configure the Order route

Add a second route with the following responsibilities:

| Route property | Required value |
|---|---|
| Route identifier | `order-service` |
| Target URI | `http://localhost:8082` |
| Public paths | `/api/orders` and `/api/orders/**` |
| Prefix transformation | Remove only the first path segment |

The order route differs from the Catalogue route.

```text
/api/orders
```

must become:

```text
/orders
```

Therefore, removing both `api` and `orders` would be incorrect. Choose the prefix count according to the target path, not according to a global convention.

## Verify the read operation

```bash
curl -i http://localhost:8080/api/orders
```

## Create the case-study order

```bash
curl -i -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"productCode":"SKU-77","quantity":1}'
```

The first created order should have the identifier:

```text
QC-1042
```

## Retrieve it

```bash
curl -i http://localhost:8080/api/orders/QC-1042
```

### Checkpoint

The gateway exposes one coherent `/api/...` namespace even though downstream services use different paths and ports.

---

# Task 3 - Add gateway filters

## 3.1 Add a default response header

Add a default filter that places the following header on gateway responses:

```text
X-QuickCart-Gateway: active
```

This header gives visible evidence that the response passed through the gateway.

Verify it with:

```bash
curl -i http://localhost:8080/api/orders
```

## 3.2 Add a Catalogue request header

Add a route-specific filter to the Catalogue route:

```text
X-Client-Channel: classroom
```

The Catalogue service echoes the received value in this response header:

```text
X-Catalog-Received-Channel
```

Verify it with:

```bash
curl -i http://localhost:8080/api/catalog/products
```

You should see both gateway-related headers.

### Design boundary

These filters are appropriate for demonstrating an edge concern. Do **not** move product availability, order validation, stock decisions, or payment rules into the gateway.

---

# Task 4 - Observe a downstream failure

Stop the **Order Service** but leave the gateway running.

Repeat:

```bash
curl -i http://localhost:8080/api/orders
```

Record the result in:

```text
docs/architecture-decisions.md
```

Include:

- the HTTP status returned by the gateway;
- the response body;
- what the client can infer;
- what the gateway cannot repair by itself.

Restart the Order service and confirm that the route recovers.

## Discussion

The API Gateway pattern provides a unified entry point. It does not automatically provide:

- service discovery;
- retries;
- circuit breaking;
- fallback behaviour;
- distributed tracing;
- authentication and authorization;
- business consistency.

Those concerns require additional decisions and, often, additional patterns.

---

# Task 5 - Complete the architecture decision notes

Complete `docs/architecture-decisions.md`.

Your notes should distinguish between:

## Appropriate gateway responsibilities

Examples include:

- routing;
- path adaptation;
- protocol or representation adaptation;
- edge authentication;
- CORS policy;
- rate limiting;
- correlation identifiers;
- response headers.

## Service responsibilities

Examples include:

- product availability;
- order quantity validation;
- pricing;
- stock reservation;
- payment authorization;
- domain-specific error decisions.

Do not add all of these mechanisms to the laboratory. Use them only to explain the boundary.

---

# Acceptance criteria

Your implementation is complete when all of the following statements are true.

## Routing

- [ ] `GET /api/catalog/products` returns the Catalogue product list through port `8080`.
- [ ] `GET /api/catalog/products/SKU-77` returns one product.
- [ ] `GET /api/orders` returns the Order service response through port `8080`.
- [ ] `POST /api/orders` creates an order.
- [ ] `GET /api/orders/QC-1042` returns the created order.

## Path transformation

- [ ] Catalogue receives `/products` rather than `/api/catalog/products`.
- [ ] Order receives `/orders` rather than `/api/orders` or `/`.
- [ ] The two routes use prefix-removal values appropriate to their own downstream paths.

## Filters

- [ ] Gateway responses contain `X-QuickCart-Gateway: active`.
- [ ] Catalogue receives `X-Client-Channel: classroom`.
- [ ] The Catalogue response confirms the received channel.

## Architecture

- [ ] Clients do not call ports `8081` or `8082` in the final verification.
- [ ] No business rule was moved into the gateway.
- [ ] A downstream failure was observed and documented.
- [ ] `docs/architecture-decisions.md` is complete.

---

# Submission

Commit the following:

1. The completed `api-gateway/src/main/resources/application.yml`.
2. The completed `docs/architecture-decisions.md`.
3. Any small README clarification needed to run the project.

Do not submit generated `target/` directories.

---

# Reflection questions

Answer briefly in the architecture-decision file or discuss in class.

1. What client coupling did the API Gateway remove?
2. What new runtime dependency did the gateway introduce?
3. Why do the Catalogue and Order routes use different prefix transformations?
4. Which gateway concerns are legitimate edge concerns?
5. Which decisions would violate service ownership if placed in the gateway?
6. How would the route configuration change if service instances received dynamic addresses?
7. What additional pattern would help prevent a slow downstream service from exhausting gateway resources?

