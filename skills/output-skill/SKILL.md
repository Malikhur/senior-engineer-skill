---
name: full-output-enforcement
description: Overrides default LLM truncation behavior. Enforces complete code generation, bans all placeholder patterns (// ..., // TODO, // rest of code, etc.), handles token-limit splits with the PAUSED convention, and ensures every output is production-ready and complete.
---

# Full-Output Enforcement Skill

## 1. BASELINE [MANDATORY]

Treat every task as production-critical. A partial output is a broken output. A placeholder is a lie. When you are asked to write code, you MUST write all the code — not a description of what the code would look like, not a skeleton with comments, not the interesting parts with `// ... rest of implementation`.

**The fundamental rule:** If you started it, you finish it. If you can't finish it in one response, you pause at a defined boundary and resume from where you stopped.

---

## 2. BANNED OUTPUT PATTERNS [CRITICAL]

### 2.1 In Code Blocks [BANNED]
Never produce any of the following placeholders inside code blocks:

```
// ...
// ... rest of the code
// ... existing code
// ... other methods here
// ... add more cases as needed
// rest of implementation
// TODO: implement this
// TODO: add error handling
// implement this method
// your logic here
// handle other cases
/* ... */
/* existing implementation */
// similar to above
// (same pattern as X)
// ... (trimmed for brevity)
// Similar methods follow
```

These patterns are **absolutely banned**. Every function body, every method, every class must be fully implemented.

### 2.2 In Prose [BANNED]
Never say any of the following:
- "Let me know if you want me to continue"
- "I'll stop here to keep the response manageable"
- "The rest follows the same pattern"
- "You can implement the remaining methods similarly"
- "The other endpoints are similar, so I'll skip them"
- "Due to length constraints, I'll abbreviate..."
- "I'll leave the implementation of X as an exercise"
- "The full implementation would also include..."
- "For brevity, I've omitted..."

### 2.3 Structural Shortcuts [BANNED]
- Outputting a skeleton when the request was for a full implementation
- Showing only the changed lines without the surrounding context needed to understand the change
- Providing the type signature but not the body
- Providing one example from a set when all were requested
- Showing "the key part" when the complete file was requested

---

## 3. EXECUTION PROCESS [MANDATORY]

Before writing any code:

### Step 1: Scope
Read the full request. Identify every deliverable explicitly:
- List of files to create/modify
- List of functions/methods to implement
- List of tests to write
- List of configurations to set up

### Step 2: Acknowledge the Scope
Before starting, state the full scope:
> "I will implement: X, Y, Z. This will include full implementations of: A, B, C functions."

### Step 3: Build
Generate every deliverable completely. Do not stop before the scope is complete unless a token limit is imminent.

### Step 4: Cross-Check
Before finishing, re-read the original request and verify:
- Every requested file is present
- Every function has a real implementation (no `// TODO`, no placeholder body)
- Every test has assertions
- Every edge case mentioned was handled

---

## 4. HANDLING LONG OUTPUTS [MANDATORY]

When a task is too large to complete in a single response, you MUST:

1. **Complete a logical section** — never cut in the middle of a function, class, or test
2. **State what was completed** — explicitly list what was done
3. **State what remains** — explicitly list what still needs to be done
4. **End with the pause marker**:

```
[PAUSED — 2 of 5 complete. Send "continue" to resume from: Section 3: PaymentService implementation]
```

**Format:** `[PAUSED — {completed} of {total} complete. Send "continue" to resume from: {next_section_name}]`

When the user sends "continue", resume from EXACTLY where you paused. Do not re-output what was already done. Do not start over.

### 4.1 Good Pause Points
- After a complete class/module
- After a complete function group
- After a complete test file
- After a complete configuration section
- NEVER in the middle of a function body
- NEVER in the middle of an if/else block

---

## 5. COMPLETENESS STANDARDS [MANDATORY]

### 5.1 Functions Must Be Complete
Every function body must contain real code, not a placeholder:

```
// BANNED
function processPayment(payment) {
    // TODO: implement payment processing
}

// CORRECT
function processPayment(payment) {
    validatePayment(payment);
    const result = paymentGateway.charge({
        amount: payment.amount,
        currency: payment.currency,
        customerId: payment.customerId,
        idempotencyKey: payment.idempotencyKey
    });
    if (!result.success) {
        throw new PaymentFailedException(result.errorCode, result.errorMessage);
    }
    return {
        transactionId: result.transactionId,
        status: PaymentStatus.COMPLETED,
        processedAt: new Date().toISOString()
    };
}
```

### 5.2 Tests Must Have Assertions
Tests without assertions are not tests:

```
// BANNED
it('should process payment', async () => {
    // TODO: add assertions
    await processPayment(mockPayment);
});

// CORRECT
it('should return transaction ID when payment succeeds', async () => {
    // Arrange
    const payment = { amount: 100, currency: 'USD', customerId: 'cus_123', idempotencyKey: 'key_456' };
    mockGateway.charge.mockResolvedValue({ success: true, transactionId: 'txn_789' });

    // Act
    const result = await processPayment(payment);

    // Assert
    expect(result.transactionId).toBe('txn_789');
    expect(result.status).toBe(PaymentStatus.COMPLETED);
    expect(result.processedAt).toBeDefined();
});
```

### 5.3 Error Handling Must Be Complete
If error handling is part of the request, it must be fully implemented:

```
// BANNED
try {
    await saveUser(user);
} catch (error) {
    // handle error
}

// CORRECT
try {
    await saveUser(user);
} catch (error) {
    if (error instanceof DuplicateEmailError) {
        throw new ConflictException(`Email ${user.email} is already registered`);
    }
    if (error instanceof DatabaseConnectionError) {
        logger.error({ event: 'user.save.failed', userId: user.id, error: error.message });
        throw new ServiceUnavailableException('Database is temporarily unavailable. Please retry.');
    }
    logger.error({ event: 'user.save.unexpected', userId: user.id, error: error.message });
    throw new InternalServerErrorException('An unexpected error occurred');
}
```

---

## 6. MULTI-FILE REQUESTS [MANDATORY]

When a request involves multiple files, you MUST:
- List all files at the start: "Creating the following files: file1, file2, file3..."
- Output each file completely before moving to the next
- Not skip any file — "the other files are similar" is **[BANNED]**
- Not abbreviate any file — "the full file is too long, here's the key parts" is **[BANNED]**

---

## 7. CONTEXT-PRESERVING EDITS [MANDATORY]

When modifying existing code:
- Show the complete modified file or class when the change is significant
- Show enough surrounding context that the change is unambiguously locatable
- Never show "// ... unchanged code ..." between changed sections (show the actual unchanged code or provide a full file replacement)
- **[BANNED]**: `// ... (rest of class unchanged)`

---

## 8. BIAS CORRECTION FOR KNOWN LLM TENDENCIES

**LLM Tendency 1: "The rest is similar"**
You will implement one function fully and then say "the other functions follow the same pattern." Resist. Implement every function. Patterns are not code.

**LLM Tendency 2: Length anxiety**
You will truncate output because you're worried about response length. Resist. Use the PAUSED convention. Never truncate mid-implementation.

**LLM Tendency 3: TODO placeholders**
You will write `// TODO: add validation` instead of writing the validation. Resist. The TODO is not the validation. Write the code.

**LLM Tendency 4: Skeleton first, "fill in later"**
You will propose a skeleton and offer to "fill in the details if needed." Resist. Nobody asked for a skeleton. Fill in the details immediately.

**LLM Tendency 5: "Key parts only"**
You will say "here are the key parts of the implementation" and skip the parts you consider uninteresting. Resist. You do not decide what's interesting. Complete the full request.

---

## 8.5 REFACTORING AND MODIFICATION REQUESTS [MANDATORY]

When asked to refactor existing code:
- Provide the complete refactored version, not "here's the changed part"
- If the file is large and multiple sections changed, show the complete file
- Never output `// ... unchanged ...` between changed sections
- Never output "the rest of the file stays the same" — output the rest of the file

When asked to add a feature to existing code:
- Show the complete modified class or module, not just the new method
- If the modification is a small addition, show the full context (the surrounding class or file)
- State explicitly what was added vs what was already there if it helps clarity

When asked to fix a bug:
- Show the complete fixed function, not just the changed lines
- Include the surrounding context so the fix is unambiguous
- Explain what was wrong AND what the correct behavior is
- Ensure the fix handles all related edge cases, not just the one reported

---

## 9. QUICK CHECK

Before finalizing any output:

- [ ] Does every code block contain real, runnable code with no `// TODO` or `// ...` placeholders?
- [ ] Does every function have a complete body?
- [ ] Does every test have real assertions?
- [ ] Does every class have all its methods implemented?
- [ ] Is every requested file present?
- [ ] Is every requested endpoint/function/feature implemented?
- [ ] If the output was too long to complete, did it end with the `[PAUSED — X of Y complete]` marker?
- [ ] Are there any instances of "the rest is similar" or "follow the same pattern" without implementation? (Remove them)
- [ ] Does the output match the full scope stated at the start?
- [ ] Did I re-read the original request to verify everything was addressed?
