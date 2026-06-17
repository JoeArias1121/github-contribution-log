# Contribution 7326: Dapr Sidecar still Ready when "failed to load components"

**Contribution Number:** [1 / 2 / 3]  
**Student:** [Joseph Arias]  
**Issue:** [[GitHub issue link](https://github.com/dapr/dapr/issues/7326)]  
**Status:** [Phase I Complete/ Phase II Complete/ Phase III / Phase IV] [In Progress / Complete]

---

## Why I Chose This Issue

This issue targets a critical control-plane vulnerability where the Dapr sidecar incorrectly reports a healthy 200 OK status via its /healthz readiness probe even when foundational infrastructure components fail to initialize on startup. In a production cloud environment, this flaw silently breaks microservice resiliency by tricking orchestrators like Kubernetes into routing live application traffic directly into a broken container. Resolving this lifecycle mismatch bridges my background in fullstack development with my goal to pivot into professional Cloud and DevOps engineering. It provides an active, high-priority arena to master strict typing, advanced error propagation, and systems logic in Go—the backbone language of modern cloud-native automation.

---

## Understanding the Issue

### Problem Description

[In your own words, what's broken or missing?]

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

---

## Reproduction Process

### Environment Setup

When working inside the official VS Code Dev Container, host-to-container directory mount permissions caused Git to flag the workspace as an unsafe repository owned by a different user. Resolved the issue by adding the repository path to the global safe directory list: git config --global --add safe.directory /workspaces/dapr. Go Toolchain Version Mismatch: * Challenge: The repository’s go.mod specifies a requirement of Go >= 1.26.4, but the baseline image bundled inside the development container was running Go 1.24.6, causing compilation blockages. Overrode the local compiler restrictions by explicitly appending the auto-download toolchain flag directly into the compilation pipeline: GOTOOLCHAIN=auto make build -B.

### Steps to Reproduce

1. Create a local test scenario that completely breaks the secret resolution phase. Inside the ./test-resources folder, create two files: local-secret-store.yaml — Points to a completely missing JSON file to trigger a file-not-found initialization error: apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: mystore
spec:
  type: secretstores.local.file
  version: v1
  metadata:
  - name: secretsFile
    value: ./test-resources/non-existent-secrets.json

    broken-component.yaml — A state store dependent on that broken secret store:
    apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisPassword
    secretKeyRef:
      name: mystore
      key: redis-password
2. Spin up the Standalone Runtime: Run the freshly compiled runtime binary pointing directly to the defective resources folder: ./dist/linux_amd64/release/daprd --app-id test-app --resources-path ./test-resources --dapr-http-port 3500
3. The sidecar's component manager registers the failure immediately and triggers an asynchronous graceful shutdown loop. However, instead of blocking execution, the boot logic moves forward. The internal web server opens port 3500 to the host, registers a false-positive status update (dapr initialized. Status: Running.), and exposes a small execution window where any external health check (/healthz) returns a successful 204 No Content code right before the fatal runtime panic executes.

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commits showing reproduction](https://github.com/dapr/dapr/compare/master...JoeArias1121:dapr:fix-issue-7326)
- **Screenshots/logs:** // 1. The core component failure occurs instantly
ERRO[0000] Failed to init component mystore (secretstores.local.file/v1): open ./test-resources/non-existent-secrets.json: no such file or directory
WARN[0000] Error processing component, daprd will exit gracefully...

// 2. The HTTP server binds to port 3500 and becomes live despite the failure
INFO[0000] HTTP server listening on TCP address: :3500
INFO[0000] HTTP server is running on port 3500

// 3. The false-positive initialization peak occurs
INFO[0000] dapr initialized. Status: Running. Init Elapsed 90ms

// 4. The asynchronous shutdown finally catches up and crashes the container
INFO[0000] Dapr is shutting down
FATA[0000] Fatal error from runtime: process component mystore error: ... open ./test-resources/non-existent-secrets.json: no such file or directory
- **My findings:** The issue stems from a total decoupling between the runtime's component initialization lifecycle and the HTTP/gRPC server loops inside pkg/runtime/runtime.go.

When a.loadComponents(ctx) faces a terminal component initialization failure, the core processor handles it by scheduling an asynchronous background exit task rather than generating a blocking synchronous error back up to initRuntime. Because execution continues uninterrupted, the HTTP server fires up and responds positively to /healthz traffic.

In production Kubernetes environments, this structural gap creates a critical race condition. If a kubelet readiness probe hits the sidecar during that microsecond timeline window between HTTP binding and the container crash, it interprets the 204 No Content status as total sidecar readiness. Kubernetes then actively routes live user traffic to a container that is actively panicking and dying, resulting in immediately dropped and rejected user requests.

---

## Solution Approach

### Analysis

The flaw exists inside pkg/runtime/runtime.go within func (a *DaprRuntime) initRuntime(...).

The runtime invokes err = a.loadComponents(ctx).

However, the underlying component processor handles initialization errors by logging an error/warning and spinning up an asynchronous background task to kill the runtime gracefully later.

Because this failure is non-blocking to the main initialization thread, initRuntime moves forward uninterrupted, executing err = a.startHTTPServer(ctx).

Since outboundHealthz has no knowledge of the background component failure, the web server answers readiness probes with a successful 204 No Content code before the process dies.

### Proposed Solution

The fix is all about introducing a gatekeeper to the health subsystem so that the web server cannot lie to Kubernetes while the house is on fire.

Instead of letting the HTTP server boot up and blindly report that everything is fine, we are going to force the health probe to actively wait for the component loading process to finish successfully.

### Implementation Plan

Using UMPIRE framework (adapted): 

**Understand:** When the Dapr sidecar (daprd) boots up with a misconfigured component (e.g., an unreachable database or a missing Kubernetes secret reference), the component initialization fails. However, instead of halting the startup sequence or marking the container as unhealthy, Dapr continues booting. It binds to the HTTP port (3500) and opens its /healthz and /readiness endpoints.

Because the health subsystem does not track the status of component initialization, the health endpoints return a false-positive 204 No Content (success) status code. In a production Kubernetes cluster, kubelet hits this endpoint during that brief window, assumes the sidecar is healthy, and marks the Pod as READY. Traffic is then routed to a container that is actively crashing or completely broken, causing dropped or failed customer requests.

Expected Behavior
If any critical component fails to initialize during startup, the Dapr sidecar's readiness probe must immediately return a failure status code (e.g., 500 Internal Server Error) to block incoming traffic, and the sidecar should gracefully exit as unready.

**Match:** Inside pkg/runtime/runtime.go, within the initRuntime function, Dapr utilizes a target-based health registry system via a.runtimeConfig.outboundHealthz.

Currently, Dapr registers and tracks individual application readiness like this:
a.runtimeConfig.outboundHealthz.AddTarget("app").Ready()

The endpoint handler calculates overall readiness by verifying that all registered targets have executed .Ready(). We can mirror this exact pattern by introducing a dedicated lifecycle target for the components subsystem itself.

**Plan:** [Step-by-step implementation plan]
1. Register Component Health Target: Early in initRuntime, we will register a new health target specifically tracking component loading status: componentTarget := a.runtimeConfig.outboundHealthz.AddTarget("components").
2. Block Readiness Until Complete: The health probe will naturally fail with an HTTP 500 status as long as this new target is not marked ready.
3. Signal Success/Failure: If components load and initialize perfectly, we will invoke componentTarget.Ready(). If the component loader returns an error, or if the processor signals a terminal initialization failure, we will leave it unready (or explicitly flag it as failed), ensuring the health endpoints immediately reflect the error state.

**Implement:** [[Link to your branch/commits as you work]](https://github.com/JoeArias1121/dapr/tree/fix-issue-7326)

**Review:** Code Formatting: Run make format to ensure strict compliance with Go styling guidelines.

Linting Requirements: Execute make lint to catch static analysis discrepancies.

**Evaluate:** Automated Testing Plan
Unit Tests: Modify pkg/runtime/runtime_test.go to construct a runtime instance that receives a mocked, broken component provider. Assert that the health endpoint returns a non-200/204 status code when queried during initialization.

Integration Tests: Run existing test pipelines using make test to verify that adding a new target to outboundHealthz does not break actor or app-channel health checks.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
