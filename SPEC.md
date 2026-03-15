# Symphony Monitor Specification

Status: Draft v1

Purpose: Define a read-only monitoring surface for Symphony runtime visibility without starting a
second orchestration loop.

Parent reference: [https://github.com/openai/symphony](https://github.com/openai/symphony)

## 1. Problem Statement

Symphony is a long-running orchestrator. Operators need to inspect what it is doing now: how many
agent runs are active, how long the monitor has been open, what token usage has accumulated, and
which issues are currently running or queued for retry.

The monitor exists to solve three operational problems:

- It exposes live operator-visible runtime status without requiring direct access to the
  orchestrator process internals.
- It prevents accidental split-brain behavior where a second "monitor" process starts its own
  scheduler and shows misleading state.
- It gives a deterministic, testable status surface that can be consumed by a terminal UI and, if
  desired, an HTTP endpoint or other tooling.

Important boundary:

- The monitor is read-only.
- The monitor does not poll the issue tracker, dispatch work, claim issues, or mutate ticket state.
- The monitor depends on Symphony orchestration state defined by the upstream reference; it does not
  replace or redefine the orchestrator.

## 2. Relationship To The Parent Symphony Reference

This specification is a focused profile of the runtime snapshot and human-readable status surface
used by [Symphony](https://github.com/openai/symphony).

Rules:

- The upstream Symphony reference remains authoritative for orchestration, retries, workspaces,
  prompt rendering, and coding-agent execution.
- This monitor spec defines how a published runtime snapshot is exposed and rendered for operators.
- If the upstream Symphony reference and this spec overlap, the upstream reference controls the
  meaning of orchestrator state, and this spec controls the read-only monitoring contract built on
  that state.

## 3. Goals and Non-Goals

### 3.1 Goals

- Read the latest published Symphony runtime snapshot from a shared surface.
- Render a human-readable operator view that reflects the real running orchestrator state.
- Support separate monitor processes or terminals without starting another orchestration loop.
- Refresh on a fixed cadence with deterministic bounded-run support for tests and CI.
- Preserve stable ordering and formatting so monitor output is predictable to humans and tests.
- Remain runtime-local and restart-safe.

### 3.2 Non-Goals

- Acting as a control plane for pausing, stopping, reprioritizing, or editing runs.
- Replacing structured logs or the persisted orchestrator state file.
- Defining a mandatory web dashboard.
- Recomputing orchestration state from tracker APIs or workspace directories.
- Becoming the source of truth for scheduler correctness.

## 4. System Overview

### 4.1 Main Components

1. `Snapshot Publisher`
   - Usually the Symphony orchestrator process.
   - Publishes a full runtime snapshot whenever operator-visible state changes or a new snapshot is
     explicitly requested.

2. `Shared Snapshot Surface`
   - Stores the latest complete snapshot in a runtime-local location.
   - Must support cross-process readers.
   - File-backed storage is the recommended baseline.

3. `Monitor Reader`
   - Reads the latest snapshot from the shared surface.
   - Returns `unavailable` when no valid snapshot exists yet.

4. `Human-Readable Monitor`
   - Renders a terminal or equivalent status view from snapshot data only.
   - Must not create its own scheduler state.

5. `Optional HTTP Status Endpoint`
   - Exposes the same snapshot payload for dashboards or automation.
   - Remains read-only.

### 4.2 Data Flow

```text
Symphony orchestrator
  -> publish full snapshot
  -> shared snapshot surface
  -> monitor reader
  -> terminal UI and/or HTTP status endpoint
```

## 5. Core Domain Model

### 5.1 Snapshot

The monitor reads a full runtime snapshot with these top-level fields:

- `captured_at` (timestamp)
- `running` (list of running rows)
- `retrying` (list of retry rows)
- `codex_totals` (aggregate token/runtime counters)
- `rate_limits` (optional map with the latest coding-agent rate limit payload)

The snapshot is a point-in-time view. Readers should treat it as immutable once loaded.

### 5.2 Running Row

Each `running` row should include:

- `issue_id`
- `issue_identifier`
- `state`
- `attempt` (nullable)
- `started_at`
- `session_id` (nullable/optional if not known yet)
- `turn_count`

### 5.3 Retry Row

Each `retrying` row should include:

- `issue_id`
- `identifier`
- `attempt`
- `due_at`
- `error` (optional)

### 5.4 Codex Totals

`codex_totals` should include:

- `input_tokens`
- `output_tokens`
- `total_tokens`
- `seconds_running`

`seconds_running` is aggregate runtime as of `captured_at`, including active sessions, matching the
upstream Symphony reference.

### 5.5 Ordering

For deterministic rendering:

- `running` rows should be sorted by `issue_identifier`, then `issue_id`.
- `retrying` rows should be sorted by `identifier`, then `issue_id`.

## 6. Snapshot Surface Requirements

### 6.1 Publication Semantics

The publisher should write whole snapshots, not partial patches.

Recommended publication moments:

- after bootstrap
- after a poll tick completes
- after dispatch creates a running entry
- after live agent session metadata changes in an operator-visible way
- after run completion, failure, retry scheduling, or release of a running entry

### 6.2 Storage Properties

Requirements:

- The snapshot surface must be runtime-local to the monitored Symphony instance.
- Cross-process readers must see only complete snapshots.
- Implementations should prefer atomic replace semantics over in-place mutation.
- Invalid or missing snapshot content should be treated as `unavailable`, not as partially valid
  state.

Recommended file-backed profile:

- Derive the status path from the configured Symphony state path or other persisted runtime root.
- Place the snapshot file next to the persisted orchestrator state file when such a file exists.
- Use a stable implementation-defined filename.

### 6.3 Error Semantics

Reader-facing error modes:

- `unavailable` when no valid snapshot is present yet
- `timeout` if a higher-level transport chooses to expose timeouts

The human-readable monitor should handle `unavailable` gracefully with a waiting state.

## 7. Human-Readable Monitor Requirements

### 7.1 Behavioral Rules

The human-readable monitor:

- must be read-only
- must not bootstrap or tick a second Symphony orchestrator
- must render from snapshot data only
- should refresh on a configurable fixed cadence
- should support bounded execution for tests (for example, max refresh cycles)
- should exit cleanly on interrupt signals

### 7.2 Loading State

If no shared snapshot is available yet, the monitor should render an explicit waiting state instead
of blank output or synthesized zero-state orchestration data.

Recommended loading content:

- monitor heading
- brief exit hint
- status section
- waiting message
- current refresh interval

### 7.3 Required Status Summary

When a snapshot is available, the monitor should display:

- `captured_at`
- running agent count
- monitor runtime
- token totals as input/output/total
- aggregate coding-agent runtime
- refresh interval

### 7.4 Required Running-Agent View

The monitor should render a running-agent view derived from `running`.

Requirements:

- Show `(none)` when there are no running rows.
- Include issue identifier, issue state, attempt, turn count, session identifier, and started time.
- Use stable formatting so repeated refreshes do not reorder identical data.

### 7.5 Width-Aware Layout

Implementations may choose presentation details, but should support both:

- a wide fixed-column table for larger terminals
- a compact per-row layout for narrow terminals

If terminal width hints are used, the threshold and fallback width should be stable and testable.

### 7.6 Logging Boundary

The monitor should avoid emitting orchestration lifecycle logs into the same terminal surface in a
way that corrupts in-place rendering.

## 8. Optional HTTP Monitoring Endpoint

If an HTTP endpoint is exposed for monitoring:

- it should return the latest snapshot payload unchanged
- it should be read-only
- it should use `GET`
- it should return `404` for unknown paths
- it should return `405` for unsupported methods on the monitoring path
- it should return a machine-readable `unavailable` response when no snapshot is present

Recommended path:

- a stable versioned read-only endpoint path

## 9. Validation and Acceptance Criteria

An implementation satisfies this spec when the following behaviors are true:

1. Starting the monitor with no shared snapshot renders a waiting view instead of starting a new
   orchestrator.
2. Publishing a snapshot from an already-running Symphony process makes that live state visible to a
   separate monitor process.
3. The rendered monitor shows running count, monitor runtime, token totals, and a running-agent
   view.
4. Empty `running` data renders a clear empty marker such as `(none)`.
5. Non-positive refresh intervals are rejected at argument/config validation time.
6. Bounded refresh mode exists so tests can run the monitor deterministically.
7. Snapshot publication is atomic enough that readers do not observe truncated JSON.
