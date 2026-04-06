---
name: api-design-senior
description: Senior API Design Engineer. Design REST APIs that conform to the REST maturity model, use HTTP semantics correctly, implement consistent error formats (RFC 7807), enforce pagination, apply proper versioning, and override the LLM tendency to misuse status codes and create inconsistent APIs.
---

# API Design Skill

## 1. API DESIGN PHILOSOPHY [MANDATORY]

An API is a contract with your consumers. Breaking changes are a betrayal of trust. Poor design becomes permanent once consumers depend on it. Design APIs as if you cannot change them after shipping — because often you can't.

**The Three Principles:**
1. **Consistency**: Every endpoint follows the same conventions — same naming, same error format, same pagination pattern
2. **Predictability**: A developer who's seen one endpoint in your API can guess how others work
3. **Backward Compatibility**: Never break existing consumers without explicit versioning and migration period

---

## 2. REST MATURITY MODEL [MANDATORY]

Target **Level 3** of the Richardson Maturity Model:

| Level | Description | You MUST |
|---|---|---|
| Level 0 | HTTP as a tunnel | NEVER build new APIs at this level |
| Level 1 | Resources | Use resource-based URLs |
| Level 2 | HTTP Verbs + Status Codes | Use verbs and status codes correctly [CRITICAL] |
| Level 3 | HATEOAS | Include `_links` in responses for discoverability |

For most APIs, Level 2 is the practical minimum. Level 3 (HATEOAS) is aspirational but valuable for public APIs.

---

## 3. RESOURCE NAMING [CRITICAL]

### 3.1 URL Structure Rules [MANDATORY]
- Resources are **nouns**, never verbs: `/users`, not `/getUsers`
- Resources are **plural**: `/users`, `/orders`, `/products`
- Hierarchical relationships expressed in path: `/users/{userId}/orders/{orderId}`
- Actions on resources use HTTP method semantics (see Section 4)
- **[BANNED]** Verbs in URLs: `/createUser`, `/deleteOrder`, `/processPayment`
- **[BANNED]** Abbreviations in URLs: `/usr`, `/ord`, `/pmt`

### 3.2 URL Formatting
- Use **kebab-case** for multi-word resources: `/order-items`, not `/orderItems` or `/order_items`
- Consistent casing throughout: choose one and never mix
- **[BANNED]** Query parameters for resource identity: `/users?id=123` — use path: `/users/123`

### 3.3 Nested Resources
- Only nest resources when the relationship is:
  - Hierarchical (child cannot exist without parent)
  - Access-controlled (child is only accessible through parent)
- Maximum nesting depth: 2 levels
  - **Acceptable**: `/users/{userId}/orders`
  - **[BANNED]**: `/users/{userId}/orders/{orderId}/items/{itemId}/details`

---

## 4. HTTP METHOD SEMANTICS [CRITICAL]

### 4.1 Method Usage Rules [MANDATORY]
| Method | Semantics | Idempotent? | Safe? |
|---|---|---|---|
| `GET` | Read resource(s) | Yes | Yes |
| `POST` | Create new resource or trigger action | No | No |
| `PUT` | Replace resource entirely (idempotent) | Yes | No |
| `PATCH` | Partial update | No | No |
| `DELETE` | Remove resource | Yes | No |

**[BANNED]** Using `GET` for operations that change state.
**[BANNED]** Using `POST` for idempotent operations when `PUT` is appropriate.
**[BANNED]** Using `DELETE` for soft deletes that don't remove the resource from GET responses.

### 4.2 Action Endpoints
When an operation doesn't map cleanly to CRUD, use an action sub-resource:
```
POST /orders/{orderId}/cancellations   ← Cancel an order
POST /users/{userId}/password-resets   ← Trigger password reset
POST /invoices/{invoiceId}/approvals   ← Approve an invoice
```
**[BANNED]**: `POST /orders/{orderId}/cancel` (verb in URL).

---

## 5. HTTP STATUS CODES [CRITICAL]

### 5.1 Correct Status Code Usage [MANDATORY]
You MUST use the correct status code for every response:

**2xx — Success:**
- `200 OK`: Successful GET, PUT, PATCH, DELETE
- `201 Created`: Successful POST that creates a resource; MUST include `Location` header
- `202 Accepted`: Asynchronous operation accepted; processing will complete later
- `204 No Content`: Successful DELETE or PATCH with no response body

**4xx — Client Error:**
- `400 Bad Request`: Malformed request syntax, invalid parameters
- `401 Unauthorized`: Authentication required or credentials invalid
- `403 Forbidden`: Authenticated but not authorized for this resource
- `404 Not Found`: Resource does not exist
- `409 Conflict`: Request conflicts with current state (duplicate, version conflict)
- `410 Gone`: Resource existed but was permanently deleted
- `422 Unprocessable Entity`: Syntactically valid but semantically invalid (failed business rules)
- `429 Too Many Requests`: Rate limit exceeded

**5xx — Server Error:**
- `500 Internal Server Error`: Unexpected server error
- `502 Bad Gateway`: Upstream service error
- `503 Service Unavailable`: Server temporarily unable to handle request
- `504 Gateway Timeout`: Upstream service timed out

### 5.2 Status Code Anti-Patterns [BANNED]
* **[BANNED]** `200 OK` with an error body — the status code and body must agree
* **[BANNED]** `200 OK` for business logic failures — use `422`, `409`, or `400`
* **[BANNED]** `500` for user errors (bad input) — use `4xx`
* **[BANNED]** `404` for authorization failures — use `403` or `404` depending on disclosure policy
* **[BANNED]** `200 OK` for async operations that haven't completed — use `202 Accepted`

---

## 6. ERROR RESPONSE FORMAT [CRITICAL]

### 6.1 RFC 7807: Problem Details [MANDATORY]
All error responses MUST conform to RFC 7807 Problem Details:

```json
{
    "type": "https://api.example.com/errors/validation-error",
    "title": "Validation Error",
    "status": 422,
    "detail": "The request body contains invalid field values.",
    "instance": "/orders/7b3a9f",
    "errors": [
        {
            "field": "email",
            "message": "Email address is not valid.",
            "code": "INVALID_EMAIL_FORMAT"
        },
        {
            "field": "quantity",
            "message": "Quantity must be greater than 0.",
            "code": "MUST_BE_POSITIVE"
        }
    ]
}
```

**[MANDATORY]** Fields: `type`, `title`, `status`, `detail`
**[MANDATORY]** Include `errors` array for validation failures
**[BANNED]** Inconsistent error response shapes across endpoints
**[BANNED]** Exposing stack traces, internal error codes, or database error messages in error responses

---

## 7. PAGINATION, FILTERING, AND SORTING [CRITICAL]

### 7.1 Pagination [MANDATORY]
**[BANNED]** Unbounded list endpoints — every list endpoint MUST be paginated.

Support both styles and choose consistently:

**Offset Pagination** (simpler, less scalable):
```
GET /orders?page=2&pageSize=20

Response:
{
    "data": [...],
    "pagination": {
        "page": 2,
        "pageSize": 20,
        "totalItems": 847,
        "totalPages": 43,
        "hasNextPage": true,
        "hasPreviousPage": true
    }
}
```

**Cursor Pagination** (preferred for large datasets and real-time feeds):
```
GET /orders?cursor=eyJpZCI6MTAwfQ==&limit=20

Response:
{
    "data": [...],
    "pagination": {
        "nextCursor": "eyJpZCI6MTIwfQ==",
        "previousCursor": "eyJpZCI6ODV9",
        "hasNextPage": true
    }
}
```

**[MANDATORY]** Default `pageSize`/`limit`: 20. Maximum: 100.

### 7.2 Filtering
- Use query parameters for filtering: `GET /orders?status=pending&customerId=123`
- Use consistent operator conventions for complex filters:
  ```
  GET /orders?createdAt[gte]=2024-01-01&createdAt[lte]=2024-12-31
  GET /orders?status[in]=pending,processing
  ```
- **[BANNED]** Filter parameters that allow arbitrary SQL or query injection

### 7.3 Sorting
- `GET /orders?sort=createdAt&order=desc`
- Support multiple sort fields: `GET /orders?sort=status,createdAt&order=asc,desc`
- **[MANDATORY]** Define and document a default sort order for every list endpoint

---

## 8. API VERSIONING [CRITICAL]

### 8.1 Versioning Strategy
**Preferred: URL path versioning**
```
/v1/users
/v2/users
```

**Acceptable: Header versioning**
```
Accept: application/vnd.api.v2+json
```

**[BANNED]** No versioning strategy — unversioned APIs cannot evolve without breaking consumers.
**[BANNED]** Query parameter versioning (`/users?version=2`) — easily missed by consumers.

### 8.2 Backward Compatibility Rules [MANDATORY]
**Non-breaking changes** (allowed without version bump):
- Adding new optional fields to responses
- Adding new optional request parameters
- Adding new endpoints

**Breaking changes** (MUST increment major version):
- Removing fields from responses
- Renaming fields
- Changing field types
- Changing status code for an existing response
- Changing URL structure

### 8.3 Deprecation Process [MANDATORY]
1. Add `Deprecation` and `Sunset` headers to deprecated endpoints
2. Document migration path in API docs
3. Maintain deprecated version for minimum 6 months (external APIs) or agreed migration window
4. Send notifications to API consumers before sunset

---

## 9. REQUEST/RESPONSE DESIGN [MANDATORY]

### 9.1 Request Body Design
- Use **camelCase** or **snake_case** consistently (choose one across the entire API)
- Mark optional vs required fields clearly in documentation
- **[BANNED]** Accepting the same field in both request body and URL path — one source of truth
- **[MANDATORY]** Validate all request fields: type, format, length, business rules

### 9.2 Response Body Design
- Wrap responses in a consistent envelope OR use flat responses — never mix:
```json
// Consistent envelope style:
{ "data": { ... }, "meta": { ... } }

// Flat style (JSON:API or direct resource):
{ "id": "123", "name": "...", "createdAt": "..." }
```
- **[BANNED]** Returning `null` for entire response bodies — use empty objects or `404`
- **[MANDATORY]** Include resource `id` in every entity response
- **[MANDATORY]** Use ISO 8601 format for all timestamps: `2024-01-15T10:30:00Z`

### 9.3 Rate Limiting Headers [MANDATORY]
Every API MUST include rate limiting headers:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 547
X-RateLimit-Reset: 1704067200
Retry-After: 60   (on 429 responses)
```

---

## 10. BANNED PATTERNS [CRITICAL]

* **[BANNED]** Verbs in URLs: `/getUser`, `/createOrder`, `/deleteProduct`
* **[BANNED]** `200 OK` for error responses — status code and body must agree
* **[BANNED]** Incorrect status codes: `404` for "wrong credentials", `500` for validation errors
* **[BANNED]** Inconsistent error response format across endpoints
* **[BANNED]** Unbounded list endpoints with no pagination
* **[BANNED]** No versioning strategy for public or consumer-facing APIs
* **[BANNED]** Breaking changes without version increment
* **[BANNED]** Exposing internal error details (stack traces, SQL errors) in API responses
* **[BANNED]** Using `GET` for state-mutating operations
* **[BANNED]** Deeply nested URLs beyond 2 resource levels
* **[BANNED]** Inconsistent naming: some fields camelCase, some snake_case in the same API
* **[BANNED]** Timestamps not in ISO 8601 format

---

## 11. BIAS CORRECTION FOR KNOWN LLM TENDENCIES

**LLM Tendency 1: Wrong status codes**
You will return `200` for errors or `404` for authorization failures. Resist. Map every response to its correct status code using the table in Section 5.

**LLM Tendency 2: Verbs in URLs**
You will generate `/api/users/createUser`. Resist. Resources are nouns. Actions come from HTTP methods.

**LLM Tendency 3: Inconsistent APIs**
You will use `camelCase` for one endpoint and `snake_case` for another, and different error formats for different controllers. Resist. Every endpoint must follow the same conventions.

**LLM Tendency 4: No pagination**
You will return all records from list endpoints without pagination. Resist. Every list endpoint must have a maximum page size and default pagination.

**LLM Tendency 5: No versioning**
You will create APIs without versioning because "we can add it later." Resist. Add versioning from the first endpoint — it's impossible to add retroactively without breaking consumers.

---

## 12. PRE-FLIGHT CHECKLIST

Before finalizing any API endpoint:

- [ ] Are all URLs resource-based nouns, not verbs?
- [ ] Does the endpoint use the correct HTTP method for the operation?
- [ ] Is the HTTP status code correct for every response case (success, validation error, not found, unauthorized, forbidden)?
- [ ] Is `200 OK` never returned for error states?
- [ ] Do all error responses conform to RFC 7807 Problem Details format?
- [ ] Are all list endpoints paginated with a documented maximum page size?
- [ ] Is a versioning strategy applied (`/v1/`, `/v2/`)?
- [ ] Are timestamps in ISO 8601 format?
- [ ] Is the response format (camelCase/snake_case) consistent with the rest of the API?
- [ ] Are rate limiting headers included in responses?
- [ ] Is the `Location` header included in `201 Created` responses?
- [ ] Are all request inputs validated with appropriate error messages?
- [ ] Does the error response expose no stack traces or internal details?
- [ ] Are breaking changes handled with a version increment?
- [ ] Is there documentation for every endpoint, field, and error code?
