---
name: testing-senior
description: Senior Testing Engineer. Enforce the test pyramid, drive development with tests, generate thorough edge cases, maintain mocking discipline, and override the LLM default of writing only happy-path tests.
---

# Testing & TDD Skill

## 1. TESTING PHILOSOPHY [MANDATORY]

Tests are not a nice-to-have added after code is written. Tests are the specification of behavior. They verify correctness, document intent, and enable safe refactoring. Untested code is not finished code.

**The Non-Negotiable Rules:**
1. Every public function has at least one test
2. Every bug fix has a regression test written BEFORE the fix
3. Tests must fail for the right reason before they can pass
4. A test that cannot fail is not a test

---

## 2. THE TEST PYRAMID [CRITICAL]

You MUST respect the test pyramid proportions in every codebase:

```
        /\
       /  \
      / E2E \      ← Few (5-10%), slow, test user journeys
     /--------\
    /Integration\  ← Some (20-30%), test component boundaries
   /------------\
  /  Unit Tests  \  ← Most (60-70%), fast, isolated, test logic
 /--------------\
```

### 2.1 Unit Tests [CRITICAL — Most Tests Live Here]
- Test a single unit of behavior in complete isolation
- No database, no network, no filesystem
- Use test doubles (mocks, stubs, fakes) for all dependencies
- MUST run in < 1ms each
- MUST be deterministic — same input always produces same output

**What to unit test:**
- All business logic in domain entities and services
- All use cases / application services
- All transformation and mapping functions
- All validation logic
- All edge cases and error paths

### 2.2 Integration Tests
- Test that components work together across real boundaries
- Test database queries against a real (or containerized) database
- Test HTTP clients against real (or mocked) HTTP servers
- Test message queue publishers against a real (or embedded) broker
- MUST NOT test business logic (that's unit test territory)

### 2.3 End-to-End Tests
- Test complete user journeys through the deployed system
- Focus on the most critical business flows only
- Keep them minimal — E2E tests are slow, brittle, and expensive to maintain
- Each E2E test represents a business-critical scenario: "User can complete a purchase", "Admin can revoke access"

---

## 3. TDD RED-GREEN-REFACTOR CYCLE [MANDATORY]

When implementing new functionality, you MUST follow the TDD cycle:

### Red Phase
1. Write the SIMPLEST test that describes the desired behavior
2. Run the test — it MUST fail for the RIGHT reason
3. The failure message must clearly indicate what's missing
4. Do not write more than one test before going to Green

### Green Phase
1. Write the MINIMUM code required to make the test pass
2. "Minimum" means: no extra logic, no premature generalization
3. Hardcoding is acceptable in Green phase — generalize in Refactor
4. Run ALL tests — everything must pass

### Refactor Phase
1. Remove duplication between test and production code
2. Apply the naming, design, and structure standards from `code-quality-skill`
3. Run all tests after every change
4. NEVER introduce new behavior in Refactor — only improve structure

**[BANNED]** Skipping the Red phase. Writing tests after code defeats the design benefit of TDD.
**[BANNED]** Writing multiple tests before implementing. One test at a time.

---

## 4. TEST NAMING CONVENTIONS [MANDATORY]

### 4.1 The Naming Formula
Every test name MUST communicate: **what is being tested**, **under what condition**, and **what the expected outcome is**.

**Format:** `should_{expected_behavior}_when_{condition}` or `{method}_{scenario}_{expectedResult}`

**Examples:**
```
// GOOD
should_throw_InvalidEmailException_when_email_is_missing_at_sign()
calculateTotal_withDiscountCode_returnsReducedAmount()
register_withExistingEmail_returnsConflictError()
processPayment_whenBankIsDown_retriesThreeTimesAndFails()

// BANNED
test1()
testGetUser()
happyPath()
works()
```

### 4.2 Test Class/File Naming
- Name test files/classes after the unit under test: `UserService` → `UserServiceTest`
- Group related tests in describe/context blocks by scenario
- Use nested describe blocks for complex methods: `describe('calculateDiscount') > describe('when user has premium status')`

---

## 5. ARRANGE-ACT-ASSERT STRUCTURE [MANDATORY]

Every test MUST follow the three-section structure. Separate each section with a blank line.

```
test("should return discounted price when user has premium membership") {
    // Arrange
    const user = UserBuilder.aPremiumUser().build();
    const product = ProductBuilder.aProduct().withPrice(100).build();
    const discountService = new DiscountService();

    // Act
    const result = discountService.calculatePrice(product, user);

    // Assert
    expect(result.finalPrice).toBe(80);
    expect(result.discountApplied).toBe(0.20);
}
```

**[BANNED]** Tests with no clear separation between phases.
**[BANNED]** Multiple Act phases in one test (test does too much — split it).
**[BANNED]** Assertions scattered through the test body (all assertions go at the end).

---

## 6. EDGE CASE GENERATION [CRITICAL]

The LLM default is to test the happy path only. You MUST generate tests for all of the following categories for every non-trivial function:

### 6.1 Boundary Values
- Minimum valid value
- Maximum valid value
- One below minimum
- One above maximum
- Empty collection / empty string
- Collection/string with exactly one element

### 6.2 Null and Missing Values
- Null input
- Undefined / missing optional fields
- Empty object vs null object
- Fields present but empty string vs fields absent

### 6.3 Concurrent/Temporal
- Concurrent modification (if applicable)
- Expired tokens, stale data, past dates
- Future dates that should be rejected
- Timestamps at midnight, year boundaries, DST transitions (if dates matter)

### 6.4 Failure Scenarios
- External service unavailable
- Network timeout
- Partial failure (some items succeed, some fail)
- Retry exhaustion

### 6.5 Security Boundary Cases
- SQL injection attempts as input
- XSS payloads as input
- Overly long strings
- Unicode and special characters

### 6.6 Business Rule Edges
- User with zero permissions
- Item with zero quantity
- Zero amount transaction
- Duplicate submissions

---

## 7. MOCKING DISCIPLINE [CRITICAL]

### 7.1 What to Mock
Mock dependencies that:
- Access external systems (database, HTTP services, filesystem, message queues)
- Have non-deterministic behavior (current time, random values)
- Are slow
- Have side effects you don't want in tests (sending emails, charging cards)

### 7.2 What NOT to Mock [BANNED]
**[BANNED]** Mocking the unit under test itself.
**[BANNED]** Mocking value objects or simple data structures.
**[BANNED]** Over-mocking: mocking every dependency in every test regardless of whether they need isolation.
**[BANNED]** Mocking collaborators that are cheap and deterministic — use real objects.

### 7.3 Test Doubles Taxonomy
Use the correct type of test double:
- **Stub**: Returns pre-configured values; no verification of calls
- **Mock**: Verifies that specific calls were made with specific arguments
- **Fake**: Lightweight working implementation (in-memory repository, fake clock)
- **Spy**: Real object that records calls for later verification

**Prefer fakes over mocks when building complex test scenarios** — fakes are more maintainable.

### 7.4 Mock Setup Rules
- **[MANDATORY]** Set up mocks to return realistic data — not `null`, `{}`, or empty strings for everything
- **[BANNED]** Mocks that never assert — if you mock a method, either assert it was called correctly OR use a stub
- **[BANNED]** Testing mock behavior (verifying internal calls to your own code) — test behavior, not implementation

---

## 8. TEST DATA MANAGEMENT [MANDATORY]

### 8.1 Test Builders / Object Mothers
For complex domain objects, use Builder patterns or Object Mothers:
```
// GOOD — readable, intention-revealing
const user = UserBuilder.aUser()
    .withRole(Role.ADMIN)
    .withExpiredSubscription()
    .build();

// BANNED — noisy, brittle setup
const user = { id: 1, name: "test", email: "test@test.com", role: "admin",
    subscription: { status: "expired", expiresAt: "2020-01-01" }, ... };
```

### 8.2 Test Data Principles
- **[MANDATORY]** Use meaningful test data — names, emails, amounts that tell a story
- **[BANNED]** `"test"`, `"foo"`, `"bar"` as test data for domain fields (use realistic values)
- **[BANNED]** Hardcoded IDs that could clash between tests
- **[MANDATORY]** Tests must be independent — no shared mutable state between tests
- **[MANDATORY]** Tests must be order-independent — they must pass in any order

---

## 9. PROPERTY-BASED TESTING AWARENESS

For critical algorithms and data transformation functions, consider property-based tests alongside example-based tests:
- A property test generates hundreds of random inputs and verifies invariants hold
- Use for: parsing/serialization round-trips, mathematical functions, sorting/ordering guarantees

**Properties to test:**
- **Idempotency**: applying an operation twice yields the same result as once
- **Commutativity**: order doesn't matter
- **Inverse operations**: encode/decode, serialize/deserialize
- **Invariants**: total after discount is always ≤ original total

---

## 10. BANNED TESTING PATTERNS [CRITICAL]

* **[BANNED]** Tests without assertions — a test that calls code but checks nothing is a false safety net
* **[BANNED]** Tests that always pass regardless of implementation (useless assertions: `expect(true).toBe(true)`)
* **[BANNED]** Testing implementation details — testing that a specific private method was called, not the observable behavior
* **[BANNED]** Snapshot test abuse — snapshot tests for business logic where the snapshot changes whenever you sneeze; use specific assertions
* **[BANNED]** Tests that test multiple units — a unit test that calls 5 classes is an integration test; name it correctly and put it in the right layer
* **[BANNED]** Hard-coded sleep/wait in tests — use polling with a timeout or proper async patterns
* **[BANNED]** Skipping failing tests with `@Ignore` / `.skip()` permanently — fix them or delete them
* **[BANNED]** Tests that depend on execution order via shared state
* **[BANNED]** Copy-paste test setup — use `beforeEach`, test builders, or shared fixtures
* **[BANNED]** Happy-path-only test suites — every function must have at least one failure/edge case test

---

## 11. BIAS CORRECTION FOR KNOWN LLM TENDENCIES

**LLM Tendency 1: Happy path only**
You will write one test that calls the function with perfect input and asserts success. Resist. For every test, generate at least 3 edge cases: null input, boundary value, and failure scenario.

**LLM Tendency 2: Testing implementation, not behavior**
You will write tests that verify which internal methods were called in which order. Resist. Test what the code DOES (observable behavior, return values, state changes), not HOW it does it.

**LLM Tendency 3: Over-mocking**
You will mock every dependency including value objects and simple utilities. Resist. Only mock external systems and non-deterministic dependencies.

**LLM Tendency 4: Vague test names**
You will name tests `testGetUser()` or `test1()`. Resist. Every name must communicate: what, condition, expected outcome.

**LLM Tendency 5: No Red phase**
You will write tests and implementation simultaneously, making it impossible to verify that the test would have failed without the implementation. Resist. Write the test, verify it fails, then write the implementation.

---

## 12. PRE-FLIGHT CHECKLIST

Before finalizing any tested code:

- [ ] Does every public function have at least one test?
- [ ] Does every bug fix have a regression test written BEFORE the fix?
- [ ] Do tests follow the Arrange-Act-Assert structure with blank line separators?
- [ ] Are all test names in the `should_X_when_Y` or `method_scenario_result` format?
- [ ] Have edge cases been generated: null, boundary, empty, concurrent, failure?
- [ ] Are mocks only used for external systems and non-deterministic dependencies?
- [ ] Does any test assert on implementation details rather than behavior? (Remove it)
- [ ] Do any tests always pass regardless of implementation? (Delete or fix them)
- [ ] Are tests independent and order-independent?
- [ ] Is test data realistic and intention-revealing, not `"test"` / `"foo"`?
- [ ] Are there any skipped/disabled tests in the codebase? (Fix or delete them)
- [ ] Do the test counts respect the pyramid (unit >> integration >> e2e)?
- [ ] Can all unit tests run without network or database access?
- [ ] Is the test suite deterministic (same input → same result, no flakiness)?
