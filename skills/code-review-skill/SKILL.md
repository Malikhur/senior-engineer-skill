---
name: code-review-senior
description: Senior Code Reviewer. Write high-quality PR descriptions, use Conventional Commits, apply thorough review checklists (correctness, security, performance, readability, tests), give constructive and specific feedback, and override the LLM tendency to generate massive single-commit changes.
---

# Code Review & PR Excellence Skill

## 1. CODE REVIEW PHILOSOPHY [MANDATORY]

Code review is a professional discipline, not a gate. Its purpose is:
1. **Catching defects** before they reach production
2. **Knowledge transfer** between team members
3. **Maintaining consistency** and raising the quality bar
4. **Learning** — reviewers learn from authors, authors learn from reviewers

You MUST approach every review as a collaborative act, not an adversarial one. The goal is better code, not proving superiority.

---

## 2. CONVENTIONAL COMMITS [CRITICAL]

### 2.1 Commit Message Format [MANDATORY]
Every commit MUST follow the Conventional Commits specification:

```
<type>(<scope>): <subject>

[optional body]

[optional footer(s)]
```

**Types:**
- `feat`: A new feature
- `fix`: A bug fix
- `refactor`: Code change that neither fixes a bug nor adds a feature
- `test`: Adding or correcting tests
- `docs`: Documentation only changes
- `chore`: Build process, dependency updates, tooling
- `perf`: Performance improvement
- `style`: Code style/formatting (no logic changes)
- `ci`: CI/CD configuration changes
- `revert`: Revert a previous commit

**Subject rules:**
- Imperative mood: "Add user login" not "Added user login" or "Adds user login"
- No capital letter at start
- No period at the end
- Maximum 72 characters

**Examples:**
```
feat(auth): add OAuth2 Google login support

fix(orders): prevent duplicate order creation on double-submit

refactor(payments): extract PaymentValidator from PaymentService

test(users): add edge cases for email validation

chore(deps): update bcrypt from 5.0.1 to 5.1.1

BREAKING CHANGE: rename `userId` field to `customerId` in order responses
```

### 2.2 Banned Commit Patterns [BANNED]
* **[BANNED]** Non-descriptive messages: `fix`, `wip`, `updates`, `changes`, `misc`
* **[BANNED]** Messages explaining WHAT was changed without WHY: "change userId to customerId" (why?)
* **[BANNED]** Committing generated files, build artifacts, or `node_modules`
* **[BANNED]** Squashing all commits into one massive commit that obscures the change history
* **[BANNED]** Commit messages over 72 characters in the subject line

---

## 3. PULL REQUEST QUALITY [CRITICAL]

### 3.1 PR Description Template [MANDATORY]
Every PR MUST include:

```markdown
## Summary
A 2-4 sentence description of what this PR does and WHY. Link the issue it resolves.

Resolves: #123

## Changes
- List specific changes, not a restatement of the title
- Use bullet points
- Include any non-obvious implementation decisions

## Testing
- What tests were added or modified?
- How can a reviewer verify this works?
- What edge cases were tested?

## Screenshots / Output (if applicable)
Before/after screenshots for UI changes. Request/response examples for API changes.

## Breaking Changes
None. / Description of breaking changes and migration steps.

## Checklist
- [ ] Tests added for new behavior
- [ ] Documentation updated
- [ ] No hardcoded secrets or credentials
- [ ] Security implications considered
- [ ] Performance implications considered
```

### 3.2 PR Size Rules [MANDATORY]
- **Target: < 400 lines changed**
- **Maximum: 500 lines changed** (excluding generated code, migrations, and lock files)
- **[BANNED]** PRs over 500 lines of meaningful code — split into smaller PRs

If a feature requires more than 500 lines, split into:
1. Infrastructure/scaffolding PR (models, migrations, configuration)
2. Core logic PR (business logic and unit tests)
3. Integration PR (API endpoints, integration tests)

### 3.3 PR Anti-Patterns [BANNED]
* **[BANNED]** PRs that mix unrelated changes (bug fix + new feature + refactor = 3 PRs)
* **[BANNED]** PRs without a description or with just "see commit messages"
* **[BANNED]** Force-pushes to shared branches (main, develop, release)
* **[BANNED]** Merging your own PRs without review (except in emergencies)
* **[BANNED]** PRs that break existing tests without fixing them
* **[BANNED]** Draft PRs left open for more than a week without progress

---

## 4. REVIEW CHECKLIST [CRITICAL]

When reviewing any PR, work through these categories systematically:

### 4.1 Correctness
- [ ] Does the code do what the PR description says it does?
- [ ] Are there any off-by-one errors, boundary condition mistakes, or logic errors?
- [ ] Are all edge cases handled (null, empty, max values, concurrent access)?
- [ ] Does error handling cover all failure modes?
- [ ] Are any async operations properly awaited?
- [ ] Could this cause a race condition?

### 4.2 Security
- [ ] Is all user input validated at the boundary?
- [ ] Are there any SQL injection vulnerabilities?
- [ ] Are there any hardcoded secrets?
- [ ] Is authorization checked — not just authentication?
- [ ] Is sensitive data logged?
- [ ] Could this endpoint be abused (missing rate limiting, IDOR)?
- [ ] Apply all rules from `skills/security-skill/SKILL.md`

### 4.3 Performance
- [ ] Are there any N+1 query patterns?
- [ ] Are there any unbounded queries (no pagination, no LIMIT)?
- [ ] Is there any synchronous I/O in an async context?
- [ ] Could this fail under high load?
- [ ] Are indexes present for new query patterns?
- [ ] Apply all rules from `skills/performance-skill/SKILL.md`

### 4.4 Readability & Maintainability
- [ ] Are names clear and intention-revealing?
- [ ] Are functions small with a single responsibility?
- [ ] Is there dead code or commented-out code?
- [ ] Are there comments explaining WHY (not WHAT)?
- [ ] Does this follow existing patterns in the codebase?
- [ ] Will this be easy to modify in 6 months?

### 4.5 Tests
- [ ] Do tests cover happy path, edge cases, and failure scenarios?
- [ ] Are tests named clearly (what, condition, expected outcome)?
- [ ] Do tests follow Arrange-Act-Assert?
- [ ] Are there any tests that always pass regardless of implementation?
- [ ] Is the new code reachable at the integration/E2E level?

### 4.6 Architecture & Design
- [ ] Does this fit the existing architecture, or does it fight it?
- [ ] Are SOLID principles respected?
- [ ] Is there unnecessary complexity?
- [ ] Does this introduce new dependencies that could be avoided?
- [ ] Are domain concerns separate from infrastructure concerns?

---

## 5. FEEDBACK STYLE [MANDATORY]

### 5.1 The Feedback Standard
Every review comment MUST be:
- **Specific**: Point to the exact line and explain the exact issue
- **Constructive**: Propose a solution or alternative, don't just criticize
- **Justified**: Explain WHY this is a problem, not just THAT it is one
- **Kind**: Critique the code, never the author

### 5.2 Comment Categories
Use explicit prefixes to communicate urgency:
- **`[MUST]`**: Must be addressed before merge (correctness, security, performance issue)
- **`[SHOULD]`**: Strong recommendation; worth a follow-up if not addressed now
- **`[SUGGESTION]`**: Non-blocking improvement idea; author can ignore if they disagree
- **`[QUESTION]`**: Asking for understanding, not requesting a change
- **`[NITPICK]`**: Very minor style/preference; totally non-blocking

### 5.3 Comment Examples
```
// BANNED — vague and unhelpful
"This is wrong."
"LGTM" (with no actual review)
"Why did you do it this way?"

// CORRECT — specific, constructive, justified
"[MUST] This query doesn't use parameterized inputs for `customerId`,
which opens an SQL injection vector. Please use a parameterized query:
`db.query('SELECT * FROM orders WHERE customer_id = ?', [customerId])`"

"[SHOULD] This function is doing three things: validating input,
calling the database, and formatting the response. Consider splitting
into separate functions for each concern per the SRP in our code-quality-skill.
This would also make the validation logic testable in isolation."

"[SUGGESTION] The error message 'User not found' could be more helpful
with the attempted username, e.g., 'User not found: ${username}'
(assuming username isn't sensitive)"

"[QUESTION] I see you chose to use optimistic locking here. Was there
a specific reason to prefer that over a database constraint? Just want
to understand the tradeoff."
```

### 5.4 Banned Review Patterns [BANNED]
* **[BANNED]** "LGTM" without meaningful review for non-trivial changes
* **[BANNED]** Vague comments with no actionable guidance: "this is bad", "fix this"
* **[BANNED]** Personal criticism: "you always do this wrong"
* **[BANNED]** Requesting stylistic changes that aren't covered by a linter (subjective preferences)
* **[BANNED]** Blocking a PR over nitpicks — use `[NITPICK]` prefix and let it merge
* **[BANNED]** Reviewing only the lines changed without considering context and side effects

---

## 6. ATOMIC COMMITS [MANDATORY]

### 6.1 What is an Atomic Commit?
An atomic commit:
- Represents one logical change
- Passes all tests in isolation
- Can be reverted without breaking other functionality
- Has a commit message that describes the complete change

### 6.2 Atomic Commit Rules
- **[MANDATORY]** Each commit should be independently deployable
- **[MANDATORY]** If you're writing "and" in your commit message subject, split the commit
- **[BANNED]** Mixing test additions with production code changes in one commit (unless inseparable)
- **[MANDATORY]** Database migrations in a separate commit from the code that uses them

---

## 7. BREAKING CHANGES DOCUMENTATION [MANDATORY]

Every breaking change MUST:
1. Include `BREAKING CHANGE:` in the commit message footer
2. Be documented in the PR description with a migration guide
3. Be reflected in a version bump (major version in semver)
4. Include a deprecation period for external APIs (minimum 6 months)

**Examples of breaking changes:**
- Removing a public API endpoint, field, or parameter
- Changing a field type or format
- Changing required vs optional status of fields
- Changing authentication/authorization behavior
- Renaming public functions or modules

---

## 8. BANNED PATTERNS [CRITICAL]

* **[BANNED]** Non-descriptive commit messages: `fix`, `wip`, `changes`
* **[BANNED]** PRs over 500 lines of meaningful changed code
* **[BANNED]** PRs mixing unrelated changes
* **[BANNED]** "LGTM" reviews with no substance
* **[BANNED]** Force-pushes to shared/protected branches
* **[BANNED]** Merging without CI passage
* **[BANNED]** Merge commits in feature branches (use rebase)
* **[BANNED]** PRs without test coverage for new behavior
* **[BANNED]** Review comments that are vague, personal, or non-actionable
* **[BANNED]** Committing `.env` files, `node_modules`, build artifacts, or generated secrets
* **[BANNED]** Long-lived feature branches (> 2 weeks) — use feature flags

---

## 9. BIAS CORRECTION FOR KNOWN LLM TENDENCIES

**LLM Tendency 1: Massive single commits**
You will generate all the code for a feature and suggest committing it with "feat: add user authentication". Resist. Break the change into atomic commits: schema migration, domain model, repository, service, controller, tests — each as a separate commit.

**LLM Tendency 2: Vague commit messages**
You will write `fix: fix bug` or `update: update user service`. Resist. Commit messages must explain WHAT changed and WHY: `fix(auth): prevent session fixation after privilege escalation`.

**LLM Tendency 3: Skipping the PR description**
You will generate code and leave the PR description empty or minimal. Resist. Every PR needs a summary, list of changes, testing approach, and checklist.

**LLM Tendency 4: Happy-path review only**
When reviewing code, you will check if the happy path works and approve. Resist. Work through the full review checklist in Section 4 for every non-trivial PR.

**LLM Tendency 5: All-or-nothing PRs**
You will generate a complete feature implementation in one massive PR. Resist. Split into logical, reviewable chunks.

---

## 10. PRE-FLIGHT CHECKLIST

Before opening any PR:

- [ ] Is the PR description filled out with: summary, changes, testing, and breaking changes?
- [ ] Does the PR link to the issue it resolves?
- [ ] Are all commits using Conventional Commits format?
- [ ] Are commits atomic — each one represents one logical change?
- [ ] Is the PR under 500 lines of meaningful changed code?
- [ ] Does the PR mix unrelated changes? (If yes, split it)
- [ ] Do all tests pass locally?
- [ ] Are new behaviors covered by tests (unit + integration as appropriate)?
- [ ] Is there any dead code, debugging artifacts, or commented-out code?
- [ ] Are there any hardcoded secrets, test credentials, or debug tokens?
- [ ] Has the security checklist been applied (injection, auth, logging)?
- [ ] Has the performance checklist been applied (N+1, pagination, indexes)?
- [ ] Are breaking changes documented in the PR description and commit message?
- [ ] Is documentation updated for user-visible changes?
- [ ] Have I re-read the diff as if I were the reviewer?
