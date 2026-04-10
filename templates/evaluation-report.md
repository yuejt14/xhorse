# Evaluation Report: Sprint {{NUMBER}}, Iteration {{ITERATION}}

## Verdict: {{PASS | PASS_WITH_WARNINGS | FAIL}}

## Score: {{X}}/{{Y}} criteria passed

## Per-Criterion Assessment

### Criterion 1: {{description}}
- **Result**: PASS | WARN | FAIL
- **Evidence**: 
- **Generator claimed**: 
- **Evaluator verified**: 

### Criterion 2: {{description}}
- **Result**: PASS | WARN | FAIL
- **Evidence**: 
- **Generator claimed**: 
- **Evaluator verified**: 

## Scope Check

**Expected files**: {{from sprint contract}}
**Actual files changed**: {{from git diff --name-only}}
**Unexpected changes**: {{list any files not in expected set}}
**Scope verdict**: PASS | WARN | FAIL

## Code Review Findings

### Critical (blocks PASS)

<!-- Issues that must be fixed before this sprint can pass -->

### Major (should fix, logged as warnings)

<!-- Significant issues that don't block the sprint but should be addressed -->

### Minor (suggestions only)

<!-- Style, readability, minor improvements -->

## Test Results

- **Existing tests**: {{PASS/FAIL count}}
- **New tests added**: {{count}}
- **Test quality**: {{Are tests meaningful or vacuous?}}

## Rework Instructions (if FAIL)

<!-- Specific, actionable items for the generator. Reference file paths and line numbers. -->

1. **Fix**: 
   - File: 
   - Issue: 
   - Expected: 

2. **Fix**: 
   - File: 
   - Issue: 
   - Expected: 
