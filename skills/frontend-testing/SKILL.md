---
name: frontend-testing
description: Browser-based UI verification via Playwright MCP. Teaches agents how to use MCP Playwright tools for frontend testing.
user-invocable: false
---

# Frontend Testing Capability

This skill provides browser-based UI verification via a Playwright MCP server.

## Applicability

This skill requires a dev server URL and MCP server name from the orchestrator's task prompt. If the task prompt does not mention frontend testing, this skill does not apply.

## Shared Rules (all roles)

1. **The dev server is managed by the orchestrator.** NEVER start, stop, restart, or modify the dev server process. It is already running at the URL provided in the orchestrator prompt.

2. **MCP tool naming.** The Playwright MCP tools are prefixed with `mcp__<server_name>__` where `<server_name>` is provided in the orchestrator prompt. Common tools:
   - `mcp__<server_name>__browser_navigate` — navigate to a URL
   - `mcp__<server_name>__browser_screenshot` — capture the current page
   - `mcp__<server_name>__browser_click` — click an element
   - `mcp__<server_name>__browser_type` — type into an input
   - `mcp__<server_name>__browser_snapshot` — get accessibility snapshot

3. **Use the dev server URL from the orchestrator prompt.** Do not hardcode `localhost:3000` or any other URL.

4. **Do not use Playwright for non-UI criteria.** Backend logic, API responses, and data processing are verified via code review and test execution, not via the browser.

5. **Do not install Playwright or browser dependencies.** The MCP server provides the browser.

## For Generators

Evaluators: skip this section.

6. **Use browser tools for UI verification, not as a substitute for tests.** After implementing a UI feature, navigate to the relevant page and verify it renders correctly. Take a screenshot to confirm. This supplements your unit/integration tests — it does not replace them.

7. **Include browser evidence in your self-assessment.** For acceptance criteria that involve UI behavior, your evidence should include what you saw via Playwright (e.g., "Navigated to /login, screenshot confirmed the login form renders with email and password fields").

8. **Handle Playwright failures gracefully.** If an MCP tool call fails (timeout, element not found, etc.), note the failure in your self-assessment under Known Issues. Do not let a Playwright failure block your implementation work. Continue implementing and committing code.

9. **If the dev server appears down, note it and continue.** Note it in your self-assessment and continue with non-UI work. Do not attempt to diagnose or fix the server process.

## For Evaluators

Generators: skip this section.

10. **Independently verify UI criteria via Playwright.** Do not trust the generator's screenshots or claims about visual output. Navigate to the page yourself, interact with it, and take your own screenshot.

11. **Grade UI criteria under the existing Correctness category.** There is no separate UI category. A UI acceptance criterion that fails is a Correctness FAIL.

12. **Include Playwright evidence in your evaluation report.** For each UI criterion, describe what you navigated to, what you observed, and whether it matches the acceptance criterion.

13. **Handle Playwright failures as evidence.** A timeout on an element that should exist per the criteria is evidence of a bug (FAIL). An infrastructure-related timeout (page never loads, unrelated to generator changes) is WARN.

14. **Distinguish "generator broke it" from "unrelated crash" when the dev server is down:**
    - Check `git diff` for changes to server configuration, entry points, or dependencies.
    - **If the generator's changes plausibly caused the crash** (modified server entry point, broke an import, syntax error): grade relevant UI criteria as **FAIL** with rework instructions.
    - **If the crash appears unrelated** (no server-related files modified): grade as **WARN** with note that UI criteria could not be verified.
    - Include the evidence for your determination.

15. **Browser tools are verification tools, not write tools.** You can navigate, snapshot, click, and type to test the UI. You still cannot create, modify, or delete any project files.
