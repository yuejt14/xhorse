# Evaluation Criteria Reference

This document defines the evaluation framework used by the evaluator agent. Each category has specific verification methods, grading rules, and hard-fail conditions.

## Grading Scale

- **PASS**: Criterion fully met. No issues found.
- **WARN**: Criterion met but with minor concerns. Does not block sprint. Logged for tracking.
- **FAIL**: Criterion not met. Sprint cannot pass. Rework required.

## Verdict Rules

- **PASS**: All criteria are PASS or WARN with no FAIL items.
- **PASS_WITH_WARNINGS**: Same as PASS but warnings exist. Sprint proceeds, warnings are logged in status.json.
- **FAIL**: Any criterion is FAIL. Rework required. Provide specific rework instructions per FAIL item.

---

## Category 1: Correctness

**What to check**:
- Logic matches the acceptance criteria in the sprint contract
- No regressions — existing tests still pass
- Data transformations produce expected output
- Control flow handles all specified scenarios

**Hard FAIL conditions**:
- Any acceptance criterion from the sprint contract is not met
- Existing tests that previously passed now fail
- Core logic produces incorrect output for specified inputs

**Verification method**:
1. Run the project's test suite
2. Review the git diff against each acceptance criterion
3. Trace data flow through new code paths
4. If the criterion says "How to verify: <command>", run that command

**Tech-stack commands**:
- Node.js: `npm test` or `npx jest` or `npx vitest`
- Python: `pytest` or `python -m unittest discover`
- Go: `go test ./...`
- Rust: `cargo test`
- Java: `./gradlew test` or `mvn test`

---

## Category 2: Test Coverage

**What to check**:
- New tests exist for new functionality
- Tests are meaningful (not vacuous — `expect(true).toBe(true)` is a FAIL)
- Tests actually exercise the acceptance criteria
- Tests cover edge cases, not just happy paths
- Tests are deterministic (no flaky tests)

**Hard FAIL conditions**:
- No tests written for new functionality
- Tests are vacuous (assertions that always pass regardless of code behavior)
- Tests don't actually test the acceptance criteria

**WARN conditions**:
- Tests exist but only cover happy paths
- Missing edge case coverage (but core paths tested)

**Verification method**:
1. List new test files: `git diff --name-only | grep -i test`
2. Read each new test — verify assertions are meaningful
3. Check that test names/descriptions map to acceptance criteria
4. Run tests and verify they pass
5. If coverage tools are configured, check branch coverage

---

## Category 3: Error Handling

**What to check**:
- All failure paths in new code are handled
- No swallowed exceptions (catch blocks that do nothing)
- Error messages are meaningful (not just "Error occurred")
- External calls (DB, API, filesystem) have error handling
- Errors propagate correctly to callers

**Hard FAIL conditions**:
- Unhandled exceptions in new code paths
- Empty catch blocks that silently swallow errors
- External calls with no error handling

**WARN conditions**:
- Error messages could be more descriptive
- Error handling exists but doesn't distinguish error types
- Missing validation on internal boundaries (acceptable if validated at entry point)

**Verification method**:
1. Search new code for try/catch, error callbacks, Result types
2. Check that catch blocks have meaningful handling
3. Review external call sites (HTTP, DB, file I/O) for error handling
4. Look for `// TODO` or placeholder error handling

---

## Category 4: Code Quality

**What to check**:
- Follows existing project conventions (naming, structure, patterns)
- No significant DRY violations (copy-pasted blocks)
- Code is readable without excessive comments
- No dead code introduced
- No scope creep — changes are within sprint contract's expected files

**Hard FAIL conditions**:
- Changed files outside the sprint contract's "Expected Files" without justification
- Large copy-pasted code blocks (3+ similar blocks)
- Introduced code that breaks existing patterns without reason

**WARN conditions**:
- Minor naming inconsistencies
- Could be more idiomatic but functionally correct
- Small amount of commented-out code

**Scope drift detection**:
1. Run `git diff --name-only` to get changed files
2. Compare against the sprint contract's "Expected Files" list
3. If files were changed that aren't in the expected set:
   - If they're clearly necessary (e.g., config files, package.json for new deps): WARN
   - If they're unrelated to the sprint scope: FAIL

**Lint/type commands** (run if available):
- Node.js: `npx eslint .` or `npx tsc --noEmit`
- Python: `ruff check .` or `mypy .`
- Go: `golangci-lint run` or `go vet ./...`
- Rust: `cargo clippy`

---

## Auto-Detection

The evaluator should detect the tech stack by checking for manifest files in the project root:

| File | Stack | Test Command | Lint Command |
|------|-------|-------------|-------------|
| `package.json` | Node.js | Read `scripts.test` or `npx jest` | `npx eslint .` |
| `pyproject.toml` | Python | `pytest` | `ruff check .` |
| `setup.py` | Python | `pytest` | `ruff check .` |
| `go.mod` | Go | `go test ./...` | `go vet ./...` |
| `Cargo.toml` | Rust | `cargo test` | `cargo clippy` |
| `build.gradle` | Java/Kotlin | `./gradlew test` | (varies) |
| `pom.xml` | Java | `mvn test` | (varies) |

If `status.json` already has `tech_stack` populated, use those commands. Otherwise, detect and report what you found.

---

## Evaluator Behavioral Rules

1. **Independence**: Do not trust the generator's self-assessment. Verify every claim independently.
2. **Skepticism**: Assume incorrect until proven otherwise. If in doubt, it's a WARN or FAIL, not a PASS.
3. **No self-talk**: If you identify a real issue, do not rationalize it away. It stays an issue.
4. **Specificity**: Every FAIL must include: the file path, what's wrong, and what the expected behavior is.
5. **Scope discipline**: Grade only what the sprint contract asks for. Don't fail a sprint for pre-existing issues.
6. **Evidence-based**: Every grade must cite evidence — a test result, a code snippet, a command output.
