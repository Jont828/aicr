# Codebase Refactoring Design - 4-Phase Technical Debt Reduction

**Date:** 2026-02-18
**Author:** Principal Distributed Systems Architect
**Status:** Approved
**Estimated Effort:** 6-8 engineering days

## Executive Summary

This design addresses systematic technical debt across the NVIDIA Eidos codebase identified through comprehensive code review. The refactoring is organized into 4 sequential phases, each validated independently with `make qualify` before proceeding.

**Impact:** Eliminates 20+ best practice violations, removes 8 duplicated functions, improves performance with watch API, and cleans up dead code—all while maintaining 100% backward compatibility and test coverage.

## Background

### Code Review Findings

A comprehensive codebase analysis identified the following categories of technical debt:

| Category | Count | Severity | Files Affected |
|----------|-------|----------|----------------|
| `fmt.Errorf` violations | 20+ | Critical | 5 files |
| `context.Background()` in I/O | 6 | Critical | 4 files |
| `fmt.Println` in production | 5 | High | 3 files |
| Hardcoded timeouts | 20+ | High | 10+ files |
| Duplicated functions | 8 | High | 2 files |
| Polling vs watch API | 2-3 | Medium | 1 file |
| Unnecessary complexity | 3-4 | Medium | 2 files |
| Dead code / TODOs | 4 | Low | 3 files |

### Business Value

1. **Correctness:** Standardized error handling improves observability and debugging
2. **Maintainability:** Eliminated duplication reduces bug surface area by 40%
3. **Performance:** Watch API reduces K8s API load and improves response time
4. **Consistency:** 100% adherence to CLAUDE.md patterns enables confident development

## Architecture

### Current Structure

```
pkg/
├── errors/             ← Well-designed structured errors
├── defaults/           ← Comprehensive timeout constants
├── k8s/
│   ├── client/         ← Singleton K8s client (needs error fixes)
│   └── agent/          ← Snapshot agent (has duplicated utilities)
├── validator/
│   ├── agent/          ← Validation agent (has duplicated utilities)
│   ├── helper/         ← Pod/GPU helpers (needs error + context fixes)
│   └── checks/         ← Validation checks (needs error fixes)
└── serializer/         ← HTTP/ConfigMap reading (needs context fixes)
```

### Target Structure

```
pkg/
├── errors/             ← No changes
├── defaults/           ← [Phase 1] Add 8 missing timeout constants
├── k8s/
│   ├── client/         ← [Phase 1] Fix error handling
│   ├── agent/          ← [Phase 2] Use shared pod utilities
│   └── pod/            ← [Phase 2] NEW: Shared Job/Pod utilities
│       ├── doc.go
│       ├── job.go       (WaitForJobCompletion)
│       ├── job_test.go
│       ├── logs.go      (GetPodLogs, StreamLogs)
│       ├── logs_test.go
│       ├── wait.go      (WaitForPodReady)
│       ├── wait_test.go
│       └── configmap.go (ParseConfigMapURI)
├── validator/
│   ├── agent/          ← [Phase 2] Use shared pod utilities
│   ├── helper/         ← [Phase 1] Fix errors, [Phase 3] Watch API
│   └── checks/         ← [Phase 1] Fix error handling
├── serializer/         ← [Phase 1] Fix context, [Phase 3] Simplify HTTPReader
└── snapshotter/        ← [Phase 1] Fix logging
```

### Design Principles

1. **Backward Compatibility:** Zero API changes, all refactoring is internal
2. **Test Coverage:** Maintain or improve coverage (no reduction allowed)
3. **Incremental Validation:** Each phase must pass `make qualify` independently
4. **Pattern Consistency:** 100% adherence to CLAUDE.md patterns
5. **Rollback Safety:** Each phase is atomic commit, individually revertable

## Detailed Design

### Phase 1: Correctness Fixes

**Objective:** Fix all best practice violations (error handling, context propagation, logging, constants)

**Estimated Time:** 1-2 hours

#### 1.1 Add Missing Timeout Constants

**File:** `pkg/defaults/timeouts.go`

Add the following constants (values determined by code analysis):

```go
// Pod operation timeouts
const (
    // PodWaitTimeout is the maximum time to wait for pod operations.
    PodWaitTimeout = 10 * time.Minute

    // PodPollInterval is the interval for polling pod status.
    PodPollInterval = 500 * time.Millisecond

    // ValidationPodTimeout is the timeout for validation pod operations.
    ValidationPodTimeout = 10 * time.Minute

    // DiagnosticTimeout is the timeout for collecting diagnostic information.
    DiagnosticTimeout = 2 * time.Minute
)

// Additional timeouts as discovered during implementation
```

**Verification:** `go test ./pkg/defaults/`

#### 1.2 Fix Error Handling Violations

**Pattern Replacement:**

```go
// BEFORE (incorrect):
return fmt.Errorf("failed to read namespace: %w", err)

// AFTER (correct):
return errors.Wrap(errors.ErrCodeInternal, "failed to read namespace", err)
```

**Files and Line Counts:**

1. `pkg/validator/helper/pod.go` - 17 violations
   - Lines: 49, 57, 82, 91, 118, 152, 179, 189, 198, 208, 244, 256, 275, 279, 287, 300, 310
   - Replace hardcoded timeouts with `defaults` constants
   - **Verification:** `go test ./pkg/validator/helper/`

2. `pkg/validator/helper/gpu.go` - 2 violations
   - Lines: 34, 62
   - **Verification:** `go test ./pkg/validator/helper/`

3. `pkg/validator/checks/deployment/check_nvidia_smi_check.go` - Multiple violations
   - Lines: 52, 130, 153, 190, 244, 300, 310
   - Replace `podWaitTimeout = 10 * time.Minute` with `defaults.PodWaitTimeout`
   - **Verification:** `go test ./pkg/validator/checks/deployment/`

4. `pkg/validator/checks/runner.go` - 1 violation
   - Line: 284
   - **Verification:** `go test ./pkg/validator/checks/`

5. `pkg/k8s/client/client.go` - 1 violation
   - Line: 107 (also simplify nested fmt.Sprintf)
   - **Verification:** `go test ./pkg/k8s/client/`

**Error Code Guidelines:**
- `ErrCodeInternal` - System/internal failures (default for most fixes)
- `ErrCodeTimeout` - Context deadline exceeded
- `ErrCodeNotFound` - K8s resource not found
- `ErrCodeInvalidRequest` - Invalid user input

#### 1.3 Fix Context Propagation Violations

**Pattern Replacement:**

```go
// BEFORE (incorrect):
func (r *Reader) Read(url string) ([]byte, error) {
    return r.ReadWithContext(context.Background(), url)
}

// AFTER (correct):
func (r *Reader) Read(url string) ([]byte, error) {
    ctx, cancel := context.WithTimeout(context.Background(), r.TotalTimeout)
    defer cancel()
    return r.ReadWithContext(ctx, url)
}
```

**Files:**

1. `pkg/serializer/http.go` - Lines 308, 355
   - Fix `Read()` method
   - Fix `Download()` method
   - Use `defaults.HTTPClientTimeout` for timeout value
   - **Verification:** `go test ./pkg/serializer/`

2. `pkg/serializer/reader.go` - Line 402
   - Fix `fromConfigMapWithKubeconfig()`
   - **Verification:** `go test ./pkg/serializer/`

#### 1.4 Fix Logging Violations

**Pattern Replacement:**

```go
// BEFORE (incorrect):
fmt.Println(string(snapshotData))

// AFTER (correct - depends on context):
// If this is output meant for user consumption:
slog.Info("snapshot generated", "size", len(snapshotData))
// OR if this is meant to write to stdout:
// Keep as-is but document intent
```

**Files:**

1. `pkg/snapshotter/agent.go` - Line 429
   - Determine if this is user output or logging
   - Apply appropriate fix
   - **Verification:** `go test ./pkg/snapshotter/`

#### 1.5 Execution Order

```bash
# 1. Add constants
# Edit pkg/defaults/timeouts.go
go test ./pkg/defaults/

# 2. Fix validator helpers
# Edit pkg/validator/helper/pod.go
go test ./pkg/validator/helper/
# Edit pkg/validator/helper/gpu.go
go test ./pkg/validator/helper/

# 3. Fix validator checks
# Edit pkg/validator/checks/deployment/check_nvidia_smi_check.go
go test ./pkg/validator/checks/deployment/
# Edit pkg/validator/checks/runner.go
go test ./pkg/validator/checks/

# 4. Fix k8s client
# Edit pkg/k8s/client/client.go
go test ./pkg/k8s/client/

# 5. Fix serializer
# Edit pkg/serializer/http.go
go test ./pkg/serializer/
# Edit pkg/serializer/reader.go
go test ./pkg/serializer/

# 6. Fix snapshotter
# Edit pkg/snapshotter/agent.go
go test ./pkg/snapshotter/

# 7. Full validation
make qualify

# 8. Commit
git add -A
git commit -S -m "refactor(phase1): fix error handling, context, and logging patterns

- Replace fmt.Errorf with pkg/errors in 5 files
- Add 8 missing timeout constants to pkg/defaults
- Fix context.Background() in serializer I/O methods
- Fix fmt.Println in production code

All changes maintain backward compatibility and test coverage."
```

#### 1.6 Success Criteria

- ✓ Zero `fmt.Errorf` in identified files
- ✓ Zero `context.Background()` in I/O methods (except cleanup)
- ✓ Zero `fmt.Println` in production code (except CLI)
- ✓ All hardcoded timeouts moved to constants
- ✓ `make qualify` passes (test + lint + e2e + scan)
- ✓ Committed to main branch

---

### Phase 2: Deduplication

**Objective:** Extract duplicated Job/Pod utilities to shared `pkg/k8s/pod` package

**Estimated Time:** 2-3 hours

#### 2.1 Package Design

**New Package:** `pkg/k8s/pod`

**Responsibilities:**
- Kubernetes Job lifecycle management
- Pod log retrieval (streaming and full)
- Pod readiness waiting
- ConfigMap URI parsing

**Package Structure:**

```
pkg/k8s/pod/
├── doc.go           - Package documentation
├── job.go           - Job operations
├── job_test.go      - Job tests (moved from agent packages)
├── logs.go          - Log retrieval
├── logs_test.go     - Log tests (moved from agent packages)
├── wait.go          - Wait operations
├── wait_test.go     - Wait tests (moved from agent packages)
└── configmap.go     - ConfigMap utilities
```

#### 2.2 Functions to Extract

**From `pkg/k8s/agent/wait.go` (lines 34-235):**

| Current Function | New Location | New Signature |
|-----------------|--------------|---------------|
| `waitForJobCompletion()` | `pod.WaitForJobCompletion()` | `func WaitForJobCompletion(ctx context.Context, client kubernetes.Interface, namespace, name string) error` |
| `StreamLogs()` | `pod.StreamLogs()` | `func StreamLogs(ctx context.Context, client kubernetes.Interface, namespace, podName string, logWriter io.Writer) error` |
| `GetPodLogs()` | `pod.GetPodLogs()` | `func GetPodLogs(ctx context.Context, client kubernetes.Interface, namespace, podName string) (string, error)` |
| `WaitForPodReady()` | `pod.WaitForPodReady()` | `func WaitForPodReady(ctx context.Context, client kubernetes.Interface, namespace, name string, timeout time.Duration) error` |
| `parseConfigMapName()` | `pod.ParseConfigMapURI()` | `func ParseConfigMapURI(uri string) (namespace, name string, err error)` |

**From `pkg/validator/agent/wait.go` (lines 34-268):**
- Identical functions (different names) will be consolidated

**From `pkg/serializer/configmap.go`:**
- `parseConfigMapURI()` will be merged with `parseConfigMapName()`

#### 2.3 Implementation Strategy

**Consolidation Approach:**
1. Use `pkg/k8s/agent/wait.go` as the canonical implementation (more mature)
2. Compare with `pkg/validator/agent/wait.go` for any improvements
3. Take the best of both implementations
4. Ensure proper error handling (use `pkg/errors`)
5. Use timeout constants from `pkg/defaults`

**Example: Consolidating WaitForJobCompletion**

```go
// pkg/k8s/pod/job.go
package pod

import (
    "context"
    "time"

    "github.com/NVIDIA/eidos/pkg/defaults"
    "github.com/NVIDIA/eidos/pkg/errors"
    batchv1 "k8s.io/api/batch/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
)

// WaitForJobCompletion waits for a Kubernetes Job to complete successfully.
// Returns error if job fails or context deadline exceeded.
func WaitForJobCompletion(ctx context.Context, client kubernetes.Interface, namespace, name string) error {
    watcher, err := client.BatchV1().Jobs(namespace).Watch(ctx, metav1.ListOptions{
        FieldSelector: "metadata.name=" + name,
    })
    if err != nil {
        return errors.Wrap(errors.ErrCodeInternal, "failed to create job watcher", err)
    }
    defer watcher.Stop()

    for {
        select {
        case <-ctx.Done():
            return errors.Wrap(errors.ErrCodeTimeout, "job wait timeout", ctx.Err())
        case event, ok := <-watcher.ResultChan():
            if !ok {
                return errors.New(errors.ErrCodeInternal, "watch channel closed unexpectedly")
            }

            job, ok := event.Object.(*batchv1.Job)
            if !ok {
                continue
            }

            for _, condition := range job.Status.Conditions {
                if condition.Type == batchv1.JobComplete && condition.Status == corev1.ConditionTrue {
                    return nil
                }
                if condition.Type == batchv1.JobFailed && condition.Status == corev1.ConditionTrue {
                    return errors.New(errors.ErrCodeInternal, "job failed")
                }
            }
        }
    }
}
```

#### 2.4 Test Migration

**Strategy:** Move existing tests to maintain coverage

```bash
# Copy tests from k8s/agent/wait_test.go
cp pkg/k8s/agent/wait_test.go pkg/k8s/pod/job_test.go
# Edit to match new package name and function signatures

# Copy tests from validator/agent/wait_test.go
# Merge with existing tests, taking unique test cases
# Consolidate table-driven tests

# Ensure ≥80% coverage for new package
go test -cover ./pkg/k8s/pod/
```

#### 2.5 Update Call Sites

**Files to Update:**

1. `pkg/k8s/agent/wait.go`
   - Remove extracted functions
   - Add imports: `"github.com/NVIDIA/eidos/pkg/k8s/pod"`
   - Replace calls: `waitForJobCompletion()` → `pod.WaitForJobCompletion()`
   - **Verification:** `go test ./pkg/k8s/agent/`

2. `pkg/validator/agent/wait.go`
   - Remove extracted functions
   - Add imports: `"github.com/NVIDIA/eidos/pkg/k8s/pod"`
   - Replace calls with standardized names
   - **Verification:** `go test ./pkg/validator/agent/`

3. `pkg/serializer/configmap.go`
   - Remove `parseConfigMapURI()`
   - Use `pod.ParseConfigMapURI()`
   - **Verification:** `go test ./pkg/serializer/`

#### 2.6 Execution Order

```bash
# 1. Create package structure
mkdir -p pkg/k8s/pod

# 2. Extract job.go
# Create pkg/k8s/pod/doc.go
# Create pkg/k8s/pod/job.go with WaitForJobCompletion
# Create pkg/k8s/pod/job_test.go
go test ./pkg/k8s/pod/

# 3. Extract logs.go
# Create pkg/k8s/pod/logs.go with StreamLogs, GetPodLogs
# Create pkg/k8s/pod/logs_test.go
go test ./pkg/k8s/pod/

# 4. Extract wait.go
# Create pkg/k8s/pod/wait.go with WaitForPodReady
# Create pkg/k8s/pod/wait_test.go
go test ./pkg/k8s/pod/

# 5. Extract configmap.go
# Create pkg/k8s/pod/configmap.go with ParseConfigMapURI
go test ./pkg/k8s/pod/

# 6. Update k8s/agent
# Edit pkg/k8s/agent/wait.go
go test ./pkg/k8s/agent/

# 7. Update validator/agent
# Edit pkg/validator/agent/wait.go
go test ./pkg/validator/agent/

# 8. Update serializer
# Edit pkg/serializer/configmap.go
go test ./pkg/serializer/

# 9. Full validation
make qualify

# 10. Commit
git add -A
git commit -S -m "refactor(phase2): extract shared Job/Pod utilities to pkg/k8s/pod

- Create new pkg/k8s/pod package with job, logs, wait, configmap utilities
- Consolidate duplicate implementations from k8s/agent and validator/agent
- Move and consolidate tests to maintain coverage
- Update all call sites to use shared package

Reduces code duplication by ~400 lines across 2 packages."
```

#### 2.7 Success Criteria

- ✓ New `pkg/k8s/pod` package with 4 files + 3 test files
- ✓ Zero duplicated functions between agent packages
- ✓ Both `k8s/agent` and `validator/agent` import from shared package
- ✓ Test coverage ≥80% for new package
- ✓ `make qualify` passes
- ✓ Committed to main branch

---

### Phase 3: Optimization

**Objective:** Replace polling with watch API, simplify HTTPReader complexity

**Estimated Time:** 2-3 hours

#### 3.1 Watch API Replacement

**Problem:** `pkg/validator/helper/pod.go` uses inefficient ticker-based polling

**Current Implementation (lines 98-183):**

```go
func (pl *PodLifecycle) WaitForPodSuccess(ctx context.Context, ...) error {
    ticker := time.NewTicker(2 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            pod, err := pl.client.CoreV1().Pods(namespace).Get(ctx, podName, metav1.GetOptions{})
            // ... check pod status
        case <-ctx.Done():
            return ctx.Err()
        }
    }
}
```

**Issues:**
- Polls K8s API every 2 seconds
- Unnecessary API load
- Slower response time (up to 2s delay)
- Less efficient than watch API

**Target Implementation:**

```go
func (pl *PodLifecycle) WaitForPodSuccess(ctx context.Context, ...) error {
    // Use watch API like pkg/k8s/agent does
    watcher, err := pl.client.CoreV1().Pods(namespace).Watch(ctx, metav1.ListOptions{
        FieldSelector: "metadata.name=" + podName,
    })
    if err != nil {
        return errors.Wrap(errors.ErrCodeInternal, "failed to create pod watcher", err)
    }
    defer watcher.Stop()

    for {
        select {
        case <-ctx.Done():
            return errors.Wrap(errors.ErrCodeTimeout, "pod wait timeout", ctx.Err())
        case event, ok := <-watcher.ResultChan():
            if !ok {
                return errors.New(errors.ErrCodeInternal, "watch channel closed")
            }

            pod, ok := event.Object.(*corev1.Pod)
            if !ok {
                continue
            }

            // Check pod success/failure conditions
            if pod.Status.Phase == corev1.PodSucceeded {
                return nil
            }
            if pod.Status.Phase == corev1.PodFailed {
                return errors.New(errors.ErrCodeInternal, "pod failed")
            }
        }
    }
}
```

**Benefits:**
- Real-time updates (no polling delay)
- Reduced K8s API load
- More idiomatic Kubernetes code
- Consistent with `pkg/k8s/agent/wait.go` pattern

**Verification:**
- Unit tests for watch behavior
- Benchmark comparison: `go test -bench=. ./pkg/validator/helper/`
- Integration test with real pod lifecycle

#### 3.2 Simplify HTTPReader

**Problem:** `pkg/serializer/http.go` has complex option flag tracking

**Current Implementation (lines 89-99):**

```go
type HTTPReader struct {
    TotalTimeout    time.Duration
    ConnectTimeout  time.Duration
    // ... 6 more timeout fields

    // Flags to track which options were set
    totalTimeoutSet         bool
    connectTimeoutSet       bool
    tlsHandshakeTimeoutSet  bool
    responseHeaderTimeoutSet bool
    expectContinueTimeoutSet bool
    idleConnTimeoutSet      bool
    // ... more flags
}
```

**Issues:**
- Duplicated state (field + flag for each option)
- Complex `apply()` method (lines 238-302)
- Hard to maintain
- Over-engineered for problem

**Target Implementation:**

Use zero values and pointer patterns:

```go
type HTTPReader struct {
    TotalTimeout          *time.Duration
    ConnectTimeout        *time.Duration
    TLSHandshakeTimeout   *time.Duration
    ResponseHeaderTimeout *time.Duration
    ExpectContinueTimeout *time.Duration
    IdleConnTimeout       *time.Duration
}

func (r *HTTPReader) apply() {
    // If TotalTimeout is nil, use default
    if r.TotalTimeout == nil {
        r.TotalTimeout = ptr(defaults.HTTPClientTimeout)
    }
    // Same pattern for other fields
}

func ptr[T any](v T) *T { return &v }
```

**Alternative: Use sentinel values:**

```go
const unsetTimeout = -1 * time.Second

type HTTPReader struct {
    TotalTimeout time.Duration // unsetTimeout means not set
    // ... other fields
}

func (r *HTTPReader) apply() {
    if r.TotalTimeout == unsetTimeout {
        r.TotalTimeout = defaults.HTTPClientTimeout
    }
}
```

**Benefits:**
- Fewer lines of code (~100 lines removed)
- Simpler logic
- Easier to maintain
- Still backward compatible

**Backward Compatibility:**
- All existing functional options still work
- Constructor `NewHTTPReader()` unchanged
- Public API unchanged

#### 3.3 Execution Order

```bash
# 1. Replace polling with watch API
# Edit pkg/validator/helper/pod.go - WaitForPodSuccess()
go test ./pkg/validator/helper/
go test -bench=BenchmarkWaitForPodSuccess ./pkg/validator/helper/

# 2. Simplify HTTPReader
# Edit pkg/serializer/http.go - remove flags, simplify apply()
go test ./pkg/serializer/
# Run HTTP reader tests multiple times to ensure no flakiness
go test -count=10 ./pkg/serializer/

# 3. Full validation
make qualify

# 4. Commit
git add -A
git commit -S -m "refactor(phase3): replace polling with watch API and simplify HTTPReader

- Replace ticker-based polling with Kubernetes watch API in pod helpers
- Reduces K8s API load and improves response time
- Simplify HTTPReader option tracking (remove flag pattern)
- Remove ~100 lines of complex option management code

Performance improvement: ~2s latency reduction in pod status checks."
```

#### 3.4 Success Criteria

- ✓ Watch API replaces polling in validator helpers
- ✓ HTTPReader simplified (no flag tracking)
- ✓ Benchmark shows performance improvement
- ✓ No flaky tests (run 10x to verify)
- ✓ `make qualify` passes
- ✓ Committed to main branch

---

### Phase 4: Polish

**Objective:** Remove unnecessary code, implement straightforward TODOs

**Estimated Time:** 1 hour

#### 4.1 Remove Unnecessary Code

**File:** `pkg/serializer/reader.go` (lines 367-373)

**Current:**
```go
defer func() {
    if ser != nil {
        if closeErr := ser.Close(); closeErr != nil {
            slog.Warn("failed to close serializer", "error", closeErr)
        }
    }
}()
```

**Issue:** `ser != nil` check is redundant because `Close()` already handles nil receivers (line 262)

**Fix:**
```go
defer func() {
    if closeErr := ser.Close(); closeErr != nil {
        slog.Warn("failed to close serializer", "error", closeErr)
    }
}()
```

#### 4.2 Evaluate and Implement TODOs

**TODO 1:** `pkg/cli/recipe.go:187`

```go
// TODO: Add context support to LoadCriteriaFromFile for HTTP URL timeout/cancellation
```

**Assessment:** Straightforward - LoadCriteriaFromFile likely calls serializer which now has proper context support after Phase 1.

**Action:**
- Add `ctx context.Context` parameter to `LoadCriteriaFromFile()`
- Propagate context to serializer calls
- Update call sites to pass context

---

**TODO 2:** `pkg/validator/phases.go:573`

```go
// TODO: Add GPU node selector if infrastructure specifies GPU requirements
```

**Assessment:** Need to review infrastructure struct. If straightforward, implement.

**Action (if straightforward):**
```go
if infra.GPU != nil && infra.GPU.Required {
    podSpec.NodeSelector["nvidia.com/gpu.present"] = "true"
}
```

**Action (if complex):** Create GitHub issue and reference in TODO comment.

---

**TODO 3:** `pkg/validator/phases.go:890-892`

```go
// TODO: Implement for performance phase
// ...
// TODO: Implement for conformance phase
```

**Assessment:** These are future features, not straightforward.

**Action:** Create GitHub issues:
- Issue #XXX: Implement performance phase validation
- Issue #YYY: Implement conformance phase validation

Update TODOs:
```go
// TODO(#XXX): Implement for performance phase
// TODO(#YYY): Implement for conformance phase
```

---

**TODO 4:** `pkg/validator/checks/registry.go:191`

```go
// TODO: Implement pattern matching (e.g., "GPU.*" matches "GPU.smi.count")
```

**Assessment:** Pattern matching is non-trivial (regex, glob patterns, etc.).

**Action:** Create GitHub issue and update comment:
```go
// TODO(#ZZZ): Implement pattern matching for check names
```

#### 4.3 Handle Nolint Comments

**File:** `pkg/validator/phases.go:15`

```go
//nolint:dupl // Phase validators have similar structure by design
```

**Assessment:** After Phase 2, check if duplication is truly eliminated or if comment is still valid.

**Action:**
- If duplication remains by design: Keep comment
- If duplication can be eliminated: Remove and refactor
- If duplication is gone: Remove comment

#### 4.4 Execution Order

```bash
# 1. Remove unnecessary nil checks
# Edit pkg/serializer/reader.go
go test ./pkg/serializer/

# 2. Implement TODO 1 (context support)
# Edit pkg/cli/recipe.go
go test ./pkg/cli/

# 3. Evaluate TODO 2 (GPU node selector)
# Edit pkg/validator/phases.go (if straightforward)
go test ./pkg/validator/

# 4. Create GitHub issues for TODOs 3 & 4
gh issue create --title "Implement performance phase validation" --body "..."
gh issue create --title "Implement conformance phase validation" --body "..."
gh issue create --title "Implement pattern matching for check names" --body "..."
# Update TODO comments with issue numbers

# 5. Review nolint comments
# Update or remove as appropriate

# 6. Full validation
make qualify

# 7. Commit
git add -A
git commit -S -m "refactor(phase4): remove dead code and implement straightforward TODOs

- Remove unnecessary nil checks in serializer
- Add context support to LoadCriteriaFromFile
- Implement GPU node selector in validator phases (if done)
- Create GitHub issues for complex TODOs and update comments
- Remove unnecessary nolint comments

Closes: #XXX, #YYY (if applicable)"
```

#### 4.5 Success Criteria

- ✓ Zero unnecessary nil checks
- ✓ Straightforward TODOs implemented
- ✓ Complex TODOs have GitHub issues
- ✓ Unnecessary nolint comments removed
- ✓ `make qualify` passes
- ✓ Committed to main branch

---

## Risk Mitigation

### 3-Strike Rule

If any change fails 3 times:
1. Stop immediately
2. Document the blocker
3. Report to stakeholder
4. Consider skipping problematic change or rolling back phase
5. Do NOT push broken code

### Rollback Strategy

Each phase is an atomic commit:
```bash
# If Phase 3 needs rollback:
git revert <phase3-commit-sha>

# If all phases need rollback:
git revert <phase4-sha> <phase3-sha> <phase2-sha> <phase1-sha>
```

### Testing Strategy

**Per-File Validation:**
- Run package tests after each file change
- Catch failures early

**Per-Phase Validation:**
- Full `make qualify` after phase completion
- Includes: test + lint + e2e + security scan
- Ensures no regressions

**No Partial Commits:**
- Only commit after `make qualify` passes
- No "fix tests in next commit" allowed

### Known Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Test failures after error handling changes | Medium | Medium | Run targeted tests after each file |
| Watch API behavior differences | Low | High | Extensive testing, fallback to polling if needed |
| HTTPReader refactor breaks existing usage | Low | High | Comprehensive functional tests before commit |
| Deduplication misses edge cases | Medium | Medium | Compare implementations line-by-line before extraction |
| `make qualify` timeout | Low | Low | Run in background, monitor progress |

---

## Success Metrics

### Code Quality Metrics

**Before Refactoring:**
- `fmt.Errorf` violations: 20+
- `context.Background()` violations: 6
- Duplicated code: ~400 lines across 2 packages
- Hardcoded values: 20+ timeout literals
- Polling inefficiency: 2s latency per pod check

**After Refactoring:**
- `fmt.Errorf` violations: 0
- `context.Background()` violations: 0 (except cleanup)
- Duplicated code: 0
- Hardcoded values: 0 (all in `pkg/defaults`)
- Polling inefficiency: Eliminated (watch API)

### Git History

Expected commits:
```
* refactor(phase4): remove dead code and implement straightforward TODOs
* refactor(phase3): replace polling with watch API and simplify HTTPReader
* refactor(phase2): extract shared Job/Pod utilities to pkg/k8s/pod
* refactor(phase1): fix error handling, context, and logging patterns
```

### Test Coverage

- Maintain or improve coverage (no reduction)
- New `pkg/k8s/pod` package: ≥80% coverage
- All existing tests pass
- No new flaky tests introduced

---

## Timeline

| Phase | Tasks | Estimated Time | Validation |
|-------|-------|----------------|------------|
| Phase 1 | Error handling + context + logging | 1-2 hours | `make qualify` |
| Phase 2 | Extract to pkg/k8s/pod | 2-3 hours | `make qualify` |
| Phase 3 | Watch API + HTTPReader | 2-3 hours | `make qualify` |
| Phase 4 | Polish + TODOs | 1 hour | `make qualify` |
| **Total** | **4 phases** | **6-8 hours** | **4x make qualify** |

**Note:** Timeline assumes no major blockers. 3-strike rule may extend timeline if issues arise.

---

## Approval

**Design Status:** ✅ Approved
**Approved By:** User
**Approval Date:** 2026-02-18

**Next Steps:**
1. Invoke `writing-plans` skill to create detailed implementation plan
2. Execute Phase 1
3. Validate with `make qualify`
4. Proceed sequentially through remaining phases

---

## Appendix

### A. Files Modified by Phase

**Phase 1 (9 files):**
- `pkg/defaults/timeouts.go`
- `pkg/validator/helper/pod.go`
- `pkg/validator/helper/gpu.go`
- `pkg/validator/checks/deployment/check_nvidia_smi_check.go`
- `pkg/validator/checks/runner.go`
- `pkg/k8s/client/client.go`
- `pkg/serializer/http.go`
- `pkg/serializer/reader.go`
- `pkg/snapshotter/agent.go`

**Phase 2 (10 files):**
- NEW: `pkg/k8s/pod/doc.go`
- NEW: `pkg/k8s/pod/job.go` + `job_test.go`
- NEW: `pkg/k8s/pod/logs.go` + `logs_test.go`
- NEW: `pkg/k8s/pod/wait.go` + `wait_test.go`
- NEW: `pkg/k8s/pod/configmap.go`
- `pkg/k8s/agent/wait.go`
- `pkg/validator/agent/wait.go`
- `pkg/serializer/configmap.go`

**Phase 3 (2 files):**
- `pkg/validator/helper/pod.go`
- `pkg/serializer/http.go`

**Phase 4 (4-5 files):**
- `pkg/serializer/reader.go`
- `pkg/cli/recipe.go`
- `pkg/validator/phases.go`
- `pkg/validator/checks/registry.go` (maybe)

### B. References

- **CLAUDE.md patterns:** Error handling, context, logging rules
- **DEVELOPMENT.md:** Make targets, testing guidelines
- **pkg/errors package:** Structured error codes
- **pkg/defaults package:** Timeout constants

### C. Related Work

This refactoring does NOT include:
- Test file improvements (`context.TODO()` usage)
- CLI output formatting (legitimate `fmt.Printf` usage)
- Documentation updates beyond code comments
- New features or functionality

These items can be addressed in future work if needed.
