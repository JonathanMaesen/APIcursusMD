# Chapter 1: Fundamentals of Web APIs

This chapter introduces the fundamental concepts, definitions, and architectural styles used to design, build, and interact with modern Web APIs. It establishes a strong theoretical foundation, focusing primarily on the REST architectural style.

---

## I. What is a Web API?

A Web API (Application Programming Interface) is a mechanism that enables two software components to communicate with each other over the World Wide Web, typically using the HTTP protocol.

> **Core Function:** Defines a set of rules and protocols for communication between a **client** (e.g., a mobile app, browser, or another server) and a **server**.
>
> **Architecture:** The API handles the request, executes server-side logic, and returns structured data (usually JSON or XML) back to the client.

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

**Representational State Transfer (REST)** is an architectural style for developing distributed applications. An API that adheres to REST principles is called a **RESTful API**.

### The 6 Constraints of REST

Roy Fielding defined six mandatory constraints that an application must satisfy to be truly RESTful.

#### 1. Client-Server Separation
The client (user interface) and the server (data storage/logic) must be separate and independent. This simplifies development, promotes portability, and improves scalability.

#### 2. Statelessness
Every request from the client to the server must contain all the information needed to understand and process the request. The server must not store any client session state between requests. This improves reliability and scalability.

#### 3. Cacheable
Responses from the server must explicitly indicate whether they are cacheable or not. This helps clients reuse data, minimizing server load and improving user perceived performance.

#### 4. Uniform Interface (The Core Constraint)
The system uses a standardized way to interact with resources, simplifying the architecture. This involves four sub-constraints:

*   **Identification of Resources:** Resources are uniquely identified via URIs (Uniform Resource Identifiers).
*   **Manipulation of Resources through Representations:** Clients manipulate resources by exchanging representations (e.g., JSON or XML) of those resources.
*   **Self-Descriptive Messages:** Each message contains enough information for the receiver to process it without relying on external context (e.g., using HTTP status codes, verbs, and media types).
*   **Hypermedia as the Engine of Application State (HATEOAS):** Resources should contain links (hypermedia) that guide the client on possible next actions.

#### 5. Layered System
The client cannot tell whether it is connected directly to the end server or to an intermediary (like a load balancer, proxy, or cache layer).

#### 6. Code-On-Demand (Optional)
The server can optionally extend the client's functionality by transferring executable code (e.g., Java applets or JavaScript). This is rarely used in modern web APIs.

---

## III. Designing a REST-Based API

REST API design revolves around resources, operations, and standard HTTP interactions.

### 1. Identifying Resources and Relationships
*   **Resources (Nouns):** Everything the API manages should be modeled as a resource, identified by a URI (e.g., `/products`, `/customers`). Resources should be descriptive nouns, not verbs.
*   **Relationships:** Define how resources relate to each other (e.g., a Customer has many Orders).
*   **Nesting:** Relationships are often represented using URI nesting: `/customers/{customerId}/orders`.

### 2. Identifying Operations and HTTP Methods
Operations on resources are mapped directly to standard HTTP methods.

| HTTP Method | CRUD Operation    | Purpose                               | Idempotent |
|-------------|-------------------|---------------------------------------|------------|
| **GET**     | Read              | Retrieve a resource or a collection.  | **Yes**    |
| **POST**    | Create            | Create a new resource.                | **No**     |
| **PUT**     | Update / Replace  | Completely replace an entire resource.| **Yes**    |
| **PATCH**   | Update / Modify   | Partially modify a resource.          | **No**     |
| **DELETE**  | Delete            | Remove a resource.                    | **Yes**    |

> **Idempotency:** An operation is idempotent if executing it multiple times yields the same result state on the server (e.g., deleting a resource multiple times is idempotent, as the result is still "deleted").

### 3. Assigning Response Codes
HTTP status codes are essential for self-descriptive messages, telling the client the result and status of the operation.

| Status Code Group | Examples                               | Purpose                                                     |
|-------------------|----------------------------------------|-------------------------------------------------------------|
| **2xx (Success)** | `200 OK`, `201 Created`, `204 No Content` | The request was successfully received, understood, and accepted. |
| **4xx (Client Error)**  | `400 Bad Request`, `401 Unauthorized`, `404 Not Found`, `409 Conflict` | The client made a mistake (invalid data, missing authentication, resource not found). |
| **5xx (Server Error)**  | `500 Internal Server Error`, `503 Service Unavailable` | The server failed to fulfill an otherwise valid request. |

**Key Codes:**
*   **`200 OK`**: Standard success response for GET, PUT, PATCH.
*   **`201 Created`**: Success response for POST, providing the URI of the newly created resource in the `Location` header.
*   **`204 No Content`**: Success response for DELETE or PUT/PATCH where the response body is intentionally empty.
*   **`400 Bad Request`**: Client error due to invalid input data (e.g., validation failed).
*   **`404 Not Found`**: The resource URI does not point to an existing resource.

---

## IV. Documenting the API

Good documentation is crucial for API adoption. It acts as a contract between the API provider and consumer, detailing endpoints, parameters, data models, and error codes.

*   **OpenAPI Specification (OAS):** A standardized, language-agnostic specification (YAML or JSON) for describing RESTful APIs.
*   **Swagger:** A set of tools built around the OAS, including Swagger UI (interactive documentation) and Swagger Codegen (client code generation).

---

## V. RPC and GraphQL APIs

While REST dominates, other architectures offer benefits for specific use cases.

### 1. What is an RPC-based API?
**Remote Procedure Call (RPC)** focuses on actions and methods rather than resources.
*   **Concept:** The client executes a function or procedure on a remote server as if it were local (e.g., `client.createUser(data)`).
*   **Naming:** Endpoints are verbs that describe the action (e.g., `/calculateTax`, `/activateAccount`).
*   **Protocols:** Includes **gRPC** (using Protocol Buffers for highly efficient, serialized communication) and traditional SOAP.
*   **When to use:** Ideal for service-to-service communication where efficiency, defined method calls, and performance are prioritized over resource discoverability.

### 2. What is a GraphQL API?
**GraphQL** is a query language for APIs where the client dictates exactly what data is needed, reducing over-fetching.
*   **Concept:** Typically exposed as a single endpoint (e.g., `/graphql`) where all requests are POSTs containing the query definition.
*   **Data Retrieval:** The client specifies the exact fields required, minimizing payload size.
*   **When to use:** Excellent for complex front-end applications (mobile apps, web apps) that need to fetch disparate data sets with a single request.

---

## VI. Real-time APIs

Real-time APIs enable immediate, continuous data exchange between client and server.

### 1. The Problem with API Polling
Traditional APIs require the client to repeatedly send requests (**poll**) to check for new data (e.g., "Are there new messages?"). This wastes resources (bandwidth, battery, server CPU) and introduces latency, as data is only received during the next poll cycle.

### 2. What is a Real-time API?
A real-time API maintains a persistent connection or uses a push mechanism to send data from the server to the client **instantly** when it becomes available, without the client needing to ask for it.

### 3. Which real-time communication technology is best for your application?

| Technology | Description | Best For... |
| :--- | :--- | :--- |
| **WebSockets** | Provides a persistent, **bidirectional** (full-duplex) communication channel over a single TCP connection. | High-frequency, two-way communication (e.g., gaming, chat apps, collaborative editing). |
| **Server-Sent Events (SSE)** | Provides a **unidirectional** channel (server-to-client) over HTTP. | Sending continuous updates to clients (e.g., stock tickers, news feeds, notifications) where the client doesn't need to send data back on the same channel. |
| **SignalR** | An ASP.NET Core library that abstracts these technologies. It automatically selects the best transport method (WebSockets, SSE, or Long Polling) based on client/server capabilities. | ASP.NET Core applications requiring real-time functionality with minimal boilerplate. |
