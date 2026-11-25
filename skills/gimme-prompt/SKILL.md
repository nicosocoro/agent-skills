---
name: gimme-prompt
description: Generates an optimized, detailed prompt for a given task. Use when the user says 'gimme-prompt', 'gimme prompt', 'give me an optimized prompt', 'optimize this prompt', or asks for a better prompt for a task they want to perform.
---

# gimme-prompt

Given a task description, produce a single optimized prompt the user can paste directly into any AI assistant.

## Instructions

1. Read the task the user provided.
2. Produce **one** optimized prompt block — nothing else before or after it.
3. The prompt must:
   - Be written in second-person imperative (e.g. "You are…", "Implement…").
   - Include explicit **context**, **constraints**, **expected output format**, and **success criteria**.
   - Be self-contained: the reader needs zero extra information to act on it.
   - Avoid filler phrases ("Please", "Could you", "I want").
   - Prefer specific, measurable language over vague adjectives.
4. ask any necessary questions before generating prompt
5. suggest improvements
6. verify retrocompatibility
7. consider consistency
8. detect possible bugs

## Output format

Return the optimized prompt inside a single markdown code block (no language tag), preceded by a one-line summary of what the prompt achieves.

**Summary:** `<one sentence — what this prompt makes the AI do>`

```
<optimized prompt here>
```

## Example

**User task:** write unit tests for a Kotlin service

**Summary:** Generates comprehensive unit tests for a Kotlin service using the project's given/when/then pattern.

```
You are a senior Kotlin engineer writing unit tests for a Kotlin service in a Spring/Gradle project.

Context:
- The service under test is [ServiceName] located at [path].
- The project uses JUnit 5, MockK, and a given/when/then helper pattern (no inline //given comments — extract to private functions).
- Existing tests live in src/test/kotlin/...

Task:
- Write unit tests covering: happy path, edge cases, and error scenarios.
- Each test method must follow the given/when/then structure using private helper functions.
- Do not add code comments unless strictly necessary.
- Mock all external dependencies with MockK.

Expected output:
- A single test file with all test cases.
- Each test named in plain English: `should <expected behavior> when <condition>`.
- All tests must compile and pass with `./gradlew test --tests "ServiceNameShould"`.
```
