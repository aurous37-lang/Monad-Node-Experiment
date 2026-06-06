# June 6, 2026 Statesync Recovery (Stale Forkpoint)

This note documents the diagnosis and recovery of the pruned Monad node after it
spent several days stuck in state sync following the June 3 power loss. It is
written for public sharing and intentionally omits private identity material,
peer details, public IP addresses, hostnames, usernames, passwords, wallet-like
values, and sensitive service arguments.

## Summary

After an abrupt shutdown around `2026-06-03 10:00 MDT`, the node's last executed
block was `78,885,141`. Over the following days the four Monad services all
reported `active`, but the node never became usable for local RPC: state sync
never completed, so HTTP RPC `8080` and WebSocket RPC `8081` stayed closed and
execution did not advance.

The root cause was a **stale consensus checkpoint**. The node was left holding a
forkpoint and validator set from **epoch `1578`**, written during the June 3
restart. The network had since advanced to **epoch `1591`**, so state sync was
chasing a target the network's state-sync providers no longer retain. The
request loop could not be served and made no progress.

The fix was to refresh the forkpoint and validator set **together** to the
current epoch-`1591` pair from the official configuration bucket, reset the
local state database for a clean sync base, restore the execution event-ring
directory, and restart the service group. State sync then re-targeted to the
live chain height and completed in roughly 42 minutes. The node is now fully
serving local RPC and WebSocket again.

## Symptoms While Stuck

Sanitized checks during the stall showed a node that looked alive but was not
making progress:

- `monad-bft`, `monad-execution`, `monad-rpc`, and `monad-otel-collector` were
  all `active`.
- Node networking was listening on TCP `8000` and the local OTEL endpoint on
  `127.0.0.1:4317`, but HTTP RPC `8080` and WebSocket RPC `8081` were not
  listening.
- A local `eth_blockNumber` request to `127.0.0.1:8080` could not connect.
- Execution showed no recent block activity; its last executed block dated to
  the June 3 outage.
- BFT was busy but spinning: filtered logs were dominated by state-sync requests
  pinned to a single prefix, against a target from June 3, together with
  frequent peer connection failures and session/ping timeouts.
- BFT memory had grown to roughly `28 GiB` while doing this unproductive work.
  System memory pressure itself was still acceptable.
- CPU temperature was low during the stall, with `Tctl` around `40 C`, because
  the node was not doing real execution work.

The key tell was that the state-sync prefix did not advance across repeated
samples and the target was several days old.

## Root Cause

Monad nodes bootstrap state from a consensus checkpoint (a forkpoint) and verify
that checkpoint's certificate against the matching validator set. State-sync
providers serve recent state only. When the local forkpoint and validator set
are far behind the current epoch, the sync target points at chain state that
providers no longer retain, so every request goes unanswered and the node never
catches up. It does not crash; it simply stalls in an active-but-non-serving
state.

In this case:

- Local forkpoint epoch: `1578` (round around `78,995,759`).
- Current network epoch at diagnosis: `1591`.
- The earlier post-power-loss repair did not resolve this, because it had no
  effect on the stale checkpoint that the node was actually using.

## Recovery Action

A privacy-safe rebuild was performed with interactive sudo. No keys, keystore
values, or identity files were read, printed, or modified.

Steps:

1. Recorded the stale forkpoint epoch and backed up the stale forkpoint and
   validator files locally before any change.
2. Stopped the Monad service group.
3. Cleared the local ledger working directory and the stale forkpoint/validator
   configuration, plus stale sockets and working artifacts.
4. Downloaded the **current** forkpoint and validator set as a matched pair from
   the official mainnet configuration bucket
   (`https://bucket.monadinfra.com/forkpoint/mainnet/forkpoint.toml` and the
   companion validators file), confirming both were epoch `1591`. Refreshing
   both together is required: the forkpoint's certificate is signed by its
   epoch's validator set, and a mismatched pair fails certificate verification
   on startup.
5. Truncated the local state database to give state sync a clean base.
6. Recreated the execution event-ring directory under the 2 MB hugepage mount,
   which restores execution events and WebSocket RPC.
7. Restarted the services in order: OTEL collector, then BFT, then execution,
   then RPC.

A safety check in the rebuild aborted before the destructive truncate step if
the freshly downloaded checkpoint did not look valid.

## Recovery Result

State sync immediately re-targeted to the live chain height (around
`79,524,945`) and began advancing across many prefixes in parallel, now
receiving provider responses and completions instead of timeouts. BFT memory
normalized from about `28 GiB` to about `17.6 GiB`. No consensus certificate
errors, asserts, or panics were observed.

State sync completed in roughly 42 minutes. Validation afterward:

- `monad-bft`, `monad-execution`, `monad-rpc`, and `monad-otel-collector` were
  all `active`.
- Expected listeners were present on TCP `8000`, `8080`, `8081`, and
  localhost-only `4317`.
- Local RPC returned block `79,531,353` for `eth_blockNumber`.
- Execution advanced through block `79,531,353` during the validation window.
- No restart loop and no certificate-verification errors.

The fast catch-up is itself informative: it confirms the multi-day stall was the
unservable stale-epoch target, not a hardware limit on sync throughput.

## Version Note

At the time of recovery, `monad-bft` `v0.14.5` had been published on GitHub
(June 2), but the Category Labs package channel still offered `0.14.4` as the
installable candidate. The operator policy is to follow the package channel
rather than hand-installing a newer source tag ahead of it, so the node stayed
on the package-current `0.14.4`. There was no separate package action required
for the recovery; the issue was the stale checkpoint, not the binary version.

## Operator Takeaways

- An `active` service set is not proof of a healthy node. After long downtime,
  confirm listeners, local RPC, execution progress, and consensus activity.
- If state sync is stuck for a long time with peer timeouts and a prefix that
  does not advance, suspect a **stale forkpoint/validator pair** first. Compare
  the local checkpoint's epoch against the current epoch published in the
  official configuration bucket before doing anything destructive.
- Always refresh the forkpoint and validator set **together**. A mismatched pair
  across an epoch boundary fails certificate verification at startup.
- State-sync providers serve recent state only, so a checkpoint that ages past
  the retention window during a long outage will not catch up on its own and
  needs a manual refresh to a current checkpoint.
- Keep recovery scripts privacy-safe: source environment files at runtime
  without tracing, and never print keys, keystore values, identities, peer
  lists, public IPs, hostnames, usernames, or sensitive arguments.
