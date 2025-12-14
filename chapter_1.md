# Chapter 1: Fundamentals of Web APIs

This chapter introduces the fundamental concepts, definitions, and architectural styles used to design, build, and interact with modern Web APIs. It establishes a strong theoretical foundation, focusing primarily on the REST architectural style.

---

## I. What is a Web API?

A Web API (Application Programming Interface) is a mechanism that enables two software components to communicate with each other over the web, typically using HTTP.

> **Core Function:** It defines a set of rules and protocols for communication between a **client** (e.g., a mobile app, browser, or another server) and a **server**. The client sends a request, the server processes it and returns a response.

```
   Client                  Server
     │                      │
     ├─ Request (HTTP) ───> │
     │                      │
     │ <── Response (JSON) ─┤
     │                      │
```

---

## II. The REST Architectural Style

**Representational State Transfer (REST)** is an architectural style for developing distributed applications. An API that adheres to REST principles is called a **RESTful API**. It is not a protocol, but a set of constraints.

### The 6 Constraints of REST

Roy Fielding defined six mandatory constraints for an application to be truly RESTful.

#### 1. Client-Server Separation
The client (user interface) and the server (data storage/logic) must be separate and independent. This allows them to be developed, deployed, and scaled independently.

#### 2. Statelessness
Every request from the client must contain all the information needed for the server to process it. The server does not store any client session state between requests.

> **Example:** Instead of relying on a session, a client sends an authentication token with every request.
>
> ```http
> GET /api/orders/123
> Authorization: Bearer <your_jwt_token>
> ```

#### 3. Cacheable
Responses must explicitly indicate whether they are cacheable. Caching helps reduce server load and improves perceived performance by allowing the client to reuse previously fetched data.

> **Example:** The `Cache-Control` header tells the client it can cache the response for 3600 seconds.
>
> ```http
> HTTP/1.1 200 OK
> Cache-Control: public, max-age=3600
> Content-Type: application/json
>
> { "id": 1, "name": "Laptop" }
> ```

#### 4. Uniform Interface (The Core Constraint)
This is the central principle of REST, simplifying the architecture by using standardized, consistent conventions. It has four sub-constraints:

*   **Identification of Resources:** Resources are identified by stable, unique URIs (Uniform Resource Identifiers). Resources are nouns, not verbs.
    *   **Good:** `/users/123`, `/products`
    *   **Bad:** `/getUserById?id=123`, `/createProduct`

*   **Manipulation of Resources through Representations:** The client interacts with a resource's *representation* (e.g., a JSON or XML document), not the resource itself.
    > **Example:** A JSON representation of a user resource.
    > ```json
    > {
    >   "id": 123,
    >   "name": "Alex Doe",
    >   "email": "alex.doe@example.com"
    > }
    > ```

*   **Self-Descriptive Messages:** Each message (request/response) contains enough information for the receiver to process it without external context. This is achieved using:
    *   **HTTP Methods:** (GET, POST, PUT, DELETE)
    *   **HTTP Status Codes:** (200, 404, 500)
    *   **Media Types:** (`Content-Type: application/json`)

*   **Hypermedia as the Engine of Application State (HATEOAS):** Responses should include links (hypermedia) that guide the client on possible next actions. This allows the client to navigate the API dynamically.
    > **Example:** A user resource response containing links to related actions.
    > ```json
    > {
    >   "id": 123,
    >   "name": "Alex Doe",
    >   "_links": {
    >     "self": { "href": "/users/123" },
    >     "orders": { "href": "/users/123/orders" },
    >     "edit": { "href": "/users/123" }
    >   }
    > }
    > ```

#### 5. Layered System
The client cannot tell if it is connected directly to the end server or to an intermediary (like a load balancer or proxy). This allows for flexible infrastructure.

#### 6. Code-On-Demand (Optional)
The server can temporarily extend the client's functionality by transferring executable code (e.g., JavaScript). This is the only optional constraint and is rarely used in modern Web APIs.

---

## III. Designing a REST-Based API

### 1. Resources and URIs
Everything is a resource, represented by a noun. URIs should be descriptive and predictable.

*   **Collection:** `/products` (All products)
*   **Single Item:** `/products/42` (Product with ID 42)
*   **Nested Collection:** `/customers/123/orders` (All orders for customer 123)

### 2. Operations and HTTP Methods

Operations are mapped to standard HTTP methods.

| HTTP Method | CRUD Operation    | Purpose                               | Idempotent | Example Request Body                               |
|-------------|-------------------|---------------------------------------|------------|----------------------------------------------------|
| **GET**     | Read              | Retrieve a resource or collection.    | Yes        | N/A                                                |
| **POST**    | Create            | Create a new resource.                | No         | `{ "name": "New Gadget", "price": 99.99 }`          |
| **PUT**     | Update / Replace  | Completely replace a resource.        | Yes        | `{ "id": 42, "name": "Updated Gadget", "price": 109.99 }` |
| **PATCH**   | Update / Modify   | Partially modify a resource.          | No         | `{ "price": 119.99 }`                              |
| **DELETE**  | Delete            | Remove a resource.                    | Yes        | N/A                                                |

*An operation is **idempotent** if running it multiple times has the same effect as running it once.*

### 3. Response Status Codes

Status codes provide clear feedback to the client.

| Status Code Group | Examples                               | Purpose                                                     |
|-------------------|----------------------------------------|-------------------------------------------------------------|
| **2xx (Success)** | `200 OK`, `201 Created`, `204 No Content` | The request was successful.                                 |
| **4xx (Client Error)**  | `400 Bad Request`, `401 Unauthorized`, `404 Not Found` | The client made a mistake (e.g., invalid data, not found). |
| **5xx (Server Error)**  | `500 Internal Server Error`            | The server failed to fulfill a valid request.               |

*   **`201 Created`** should be returned after a successful `POST`, with a `Location` header pointing to the new resource: `Location: /products/43`.
*   **`204 No Content`** is often used for successful `DELETE` requests.

---

## IV. API Documentation (OpenAPI/Swagger)

Good documentation is a contract between the API provider and consumer. The **OpenAPI Specification** (formerly Swagger) is the industry standard for describing REST APIs.

> **Example:** A small snippet of an `openapi.yaml` file.
>
> ```yaml
> openapi: 3.0.0
> info:
>   title: Simple Products API
>   version: 1.0.0
> paths:
>   /products/{productId}:
>     get:
>       summary: Get a product by ID
>       parameters:
>         - name: productId
>           in: path
>           required: true
>           schema:
>             type: integer
>       responses:
>         '200':
>           description: A single product.
>           content:
>             application/json:
>               schema:
>                 $ref: '#/components/schemas/Product'
> ```

---

## V. Alternative API Architectural Styles

### 1. Remote Procedure Call (RPC)
RPC focuses on **actions** (verbs) rather than resources. The client executes a function on a remote server.

> **Example:** Endpoints are named after actions.
>
> ```
> POST /sendPasswordResetEmail
> Body: { "email": "user@example.com" }
> ```
> **gRPC** is a modern, high-performance RPC framework from Google.

### 2. GraphQL
A query language for APIs. The client requests exactly the data it needs in a single request, preventing over-fetching.

> **Example:** A single `POST` request to `/graphql` can fetch complex data.
>
> ```graphql
> query {
>   user(id: "123") {
>     name
>     email
>     orders(last: 3) {
>       orderId
>       total
>     }
>   }
> }
> ```

### 3. Real-time APIs (WebSockets & SSE)
Used for continuous, low-latency data exchange.

*   **WebSockets:** A persistent, **bidirectional** communication channel. Great for chat apps, gaming, and collaborative tools.
*   **Server-Sent Events (SSE):** A **unidirectional** channel where the server pushes updates to the client. Perfect for notifications, live scores, or stock tickers.

> **Example:** Client-side JavaScript for SSE.
>
> ```javascript
> const eventSource = new EventSource('/api/notifications');
> eventSource.onmessage = (event) => {
>   const notification = JSON.parse(event.data);
>   console.log('New notification:', notification);
> };
> ```
