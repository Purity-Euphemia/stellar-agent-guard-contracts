# Policy lifecycle walkthrough: install → pause → unfreeze → revoke → reinstall

This walkthrough demonstrates the operator-visible state changes and side
effects that occur when an admin installs, pauses, unfreezes, revokes, and
reinstalls a policy. Its goal is to bridge the gap between the individual
function reference pages and the real sequence an operator performs during
day-to-day operations.

Cross-references
- See SPEC §7 for the `PolicyConfig` fields and encoding.
- Pause behavior is implemented as a `set_policy()` write that replaces the
  whole policy (it resets the rolling window and re-arms the dead-man
  clock) — see issue #289 for discussion. A dedicated `pause()`/`resume()`
  API is proposed in issue #290.

Worked example policy (use these numbers consistently below):

- `per_tx_cap`: 1000
- `window_cap`: 150
- `window_secs`: 60
- `dms_grace_secs`: 60
- `assets`: [SAC_TOKEN]
- `recipients`: [TRUSTED_RECIPIENT]

All commands below are illustrative. Replace `GUARD`, `ADMIN`, `SAC_TOKEN`,
`TRUSTED_RECIPIENT` and network flags with your real values.

1) Install a fresh policy — `set_policy`

What you do

```
stellar contract invoke --id GUARD --network testnet --source-account ADMIN --send=yes -- set_policy --config '{ "per_tx_cap":"1000", "window_cap":"150", "window_secs":60, "dms_grace_secs":60, "assets":["SAC_TOKEN"], "recipients":["TRUSTED_RECIPIENT"] }'
```

Resulting state and side effects
- `has_policy`: true
- Rolling window: reset to empty (all previous window spend is forgotten)
- Dead-man switch: re-armed (the install gives a fresh grace window)
- `paused`: false (unless you included `paused: true` in the config)

Why it matters
- `set_policy()` is a full write — it replaces the stored `PolicyConfig`, resets
  the rolling accounting and re-arms the DMS. If you intended to only pause the
  policy, using `set_policy(paused=true)` is *not* a no-op: it also resets the
  window and re-arms the clock (see issue #289).

2) Pause the policy (admin wants an emergency freeze without revoking)

What you do

```
stellar contract invoke --id GUARD --network testnet --source-account ADMIN --send=yes -- set_policy --config '{ ... , "paused": true }'
```

Resulting state and side effects
- `paused`: true — every authorization is blocked while paused
- Rolling window: reset (because `set_policy()` replaces the policy)
- Dead-man switch: re-armed (policy install semantics apply)

Why it matters
- Because `set_policy()` replaces the whole policy, pausing via `set_policy(paused=true)` also resets the rolling window and restarts the DMS grace. If your goal is to *temporarily* suspend agent activity but keep window accounting intact, note this behaviour and the open discussion in issue #289/#290.

3) Agent stops heartbeats → account self-freezes (dead-man)

What happens (automated)
- If the agent stops sending `heartbeat()` calls and `dms_grace_secs` elapses, the
  contract will mark the account as `heartbeat_expired` (effectively a self-freeze).

Observed state
- `heartbeat_expired`: true
- `auth_checked` for agent-signed transactions: blocked with `heartbeat_expired`

Why it matters
- A self-freeze cannot be unilaterally reversed by the agent. Only an admin can
  call `unfreeze()` to clear `AdminFrozen` and/or re-arm the DMS (see next step).

4) Admin unfreezes — `unfreeze`

What you do

```
stellar contract invoke --id GUARD --network testnet --source-account ADMIN --send=yes -- unfreeze
```

Resulting state and side effects
- `AdminFrozen` cleared (if it was set)
- `LastHeartbeat` set to now (the call *re-arms* the dead-man clock). The emitted `EventUnfrozen` includes `rearmed_dms: bool` to indicate whether `LastHeartbeat` was updated.

Why it matters
- `unfreeze()` both clears the admin freeze and re-arms the dead-man switch. If the DMS grace had already elapsed, this call restarts the liveness clock; monitoring and audit logs should record `rearmed_dms` so you can tell whether the call actually rearmed the clock or merely cleared an admin freeze.

5) Revoke the policy — `revoke_policy`

What you do

```
stellar contract invoke --id GUARD --network testnet --source-account ADMIN --send=yes -- revoke_policy
```

Resulting state and side effects
- `has_policy`: false
- Rolling window: removed/cleared (no active window accounting)
- Account moves to default-deny: any agent-signed authorization is rejected (no policy)

Why it matters
- `revoke_policy()` is the irreversible administrative action that places the account into default-deny until a new policy is installed. It clears the rolling window, so any previous spend history is gone.

6) Reinstall a policy (different caps) — `set_policy`

What you do

```
stellar contract invoke --id GUARD --network testnet --source-account ADMIN --send=yes -- set_policy --config '{ "per_tx_cap":"500", "window_cap":"100", "window_secs":60, "dms_grace_secs":60, "assets":["SAC_TOKEN"], "recipients":[] }'
```

Resulting state and side effects
- New `PolicyConfig` installed
- Rolling window: reset (fresh, empty window) — previous revoked window does not carry forward
- Dead-man switch: re-armed at install time

Why it matters
- Because `set_policy()` resets the window and re-arms the DMS, reinstalling a policy always starts with a clean accounting slate and a fresh grace period for the agent to resume heartbeats.

Summary table (side effects)

| Action | Resets rolling window? | Re-arms DMS? |
|---|---:|---:|
| `set_policy()` (install / pause / reinstall) | Yes | Yes |
| `unfreeze()` | No (but clears AdminFrozen) | Yes (sets LastHeartbeat = now) |
| `revoke_policy()` | Yes (clears) | n/a |

Recommendations for operators

- If you need to pause activity without resetting accounting, avoid `set_policy()` unless you accept the window reset. Track the discussion in issue #289 and consider coordinated changes if a true pause-only semantics is required.
- When re-installing policy after a revoke, expect the window to be empty — plan monitoring and alerts accordingly.
- Use the emitted events (`EventPolicySet`, `EventUnfrozen`, `EventPolicyRevoked`) and `status()`/`check_detailed()` reads to confirm the observed state after each administrative step.

See also

- `status()` and `check_detailed()` reads for operator verification.
- docs/policy-templates.md for example policies referenced in tests.
