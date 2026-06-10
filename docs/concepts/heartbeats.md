# Heartbeats

Gas Town has **three distinct heartbeat stores**. They have different writers,
different readers, and different staleness thresholds. Refreshing one does NOT
refresh the others — confusing them causes false "stuck agent" escalations
(see hq-qxl9: a Deacon refreshed its session heartbeat for 50 minutes while
the file-mtime store aged past threshold, producing an escalation storm).

## The three stores

### 1. Deacon heartbeat file — `<townRoot>/deacon/heartbeat.json`

- **Written by:** `gt deacon heartbeat [action]` → `deacon.Touch()` /
  `deacon.TouchWithAction()` (`internal/deacon/heartbeat.go`). Since hq-qxl9,
  plain `gt heartbeat` also touches this file when `GT_ROLE=deacon`.
- **Read by:** the stuck-agent-dog plugin (stats the file **mtime**, escalates
  HIGH at >1200s) and the Go daemon (`deacon.ReadHeartbeat`, parses the
  `timestamp` field; thresholds 5m stale / 20m very-stale → poke).
- **Also touches:** the legacy `deacon/.deacon-heartbeat` mtime file for old
  shell scripts.

### 2. Session heartbeat (per-session state store)

- **Written by:** `gt heartbeat [--state=working|idle|exiting|stuck]` →
  `polecat.TouchSessionHeartbeatWithState()`. Requires `GT_SESSION`.
- **Read by:** the Witness, which reads the self-reported state instead of
  inferring liveness from timers (ZFC: gt-3vr5). This is the store polecats
  refresh.

### 3. Agent-bead label — `heartbeat:<EPOCH>` on the agent bead (e.g. `hq-deacon`)

- **Written by:** `gt mol await-signal` on each timeout/signal wake
  (`updateAgentHeartbeat` in `internal/cmd/molecule_await_signal.go`). A
  label rewrite is used because `bd agent heartbeat` was never shipped
  (steveyegge/beads#2828). Since hq-qxl9, the Deacon's `gt heartbeat` also
  syncs this label when it is >10 minutes stale.
- **Read by:** Witness second-order monitoring ("who watches the watchers"):
  Witnesses check the Deacon's bead activity and alert the Mayor if it looks
  unresponsive (>5 minutes per the patrol formula).
- **Gotcha:** a session that never reaches `await-signal` (handoff churn,
  session limits, one very long patrol turn) leaves this label stale for
  hours even though the agent is healthy.

## Rules of thumb

- **Deacon sessions:** `gt heartbeat` now refreshes all relevant stores
  (session + file + throttled bead label). On older binaries, run
  `gt deacon heartbeat` explicitly each patrol cycle.
- **Polecats / Witness / Refinery:** `gt heartbeat` (session store) is the
  one that matters.
- **Monitoring scripts:** never declare an agent stuck from a single store.
  Cross-check tmux session activity (`tmux display-message -p
  '#{window_activity}'`) before escalating — a live session with a stale
  store is *heartbeat-write divergence*, not a stuck agent. The
  stuck-agent-dog plugin does this since hq-qxl9.
