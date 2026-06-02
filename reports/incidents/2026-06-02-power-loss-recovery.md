# June 2, 2026 Power-Loss Recovery

This note documents a local recovery after an unexpected power outage on the
mini PC running the pruned Monad node. It is written for public sharing and
intentionally omits private identity material, peer details, public IP
addresses, hostnames, usernames, passwords, wallet-like values, and sensitive
service arguments.

## Summary

The machine rebooted after a power loss at about `2026-06-02 15:50 MDT`.
The PC itself came back cleanly: system memory, swap, disk usage, temperatures,
and failed-unit checks were acceptable. The Monad services also started, but
the node was not immediately usable for local RPC because state sync had not
completed.

A controlled restart of the Monad service group was performed at about
`2026-06-02 16:16 MDT`. After roughly two more minutes of state sync, local RPC
and WebSocket RPC opened again, execution resumed, and BFT logs showed normal
commit/vote activity.

## Initial Post-Outage State

Shortly after boot:

- `monad-bft`, `monad-execution`, `monad-rpc`, and `monad-otel-collector` were
  all active.
- Node networking was listening on TCP `8000`.
- The local OTEL endpoint was listening on `127.0.0.1:4317`.
- HTTP RPC `8080` and WebSocket RPC `8081` were not listening yet.
- A local `eth_blockNumber` request could not connect to `127.0.0.1:8080`.
- `monad-rpc` reported `Waiting for statesync to complete`.
- Execution had not yet shown recent block execution.
- BFT was alive, but repeated blocksync retries showed that it was not yet
  advancing normally.

PC health checks at that stage were not alarming:

- Memory was healthy, with roughly `34 GiB` available and no swap in use.
- Disk usage on the root filesystem was about `23%`.
- CPU temperature was cool shortly after boot, around `39 C`.
- No failed systemd units were listed.

## Recovery Action

After the node remained in the same non-serving state, the Monad services were
restarted in a controlled way. No destructive data action was taken, and no
node database or identity files were modified.

Services restarted:

- `monad-otel-collector`
- `monad-execution`
- `monad-rpc`
- `monad-bft`

Immediately after restart, the services were active again and RPC still waited
briefly for state sync. This was expected during the short post-restart recovery
window.

## Recovery Result

At about `2026-06-02 16:17 MDT`, `monad-rpc` reported that state sync had
completed and started the RPC servers.

Validation after recovery:

- `monad-bft`, `monad-execution`, `monad-rpc`, and `monad-otel-collector` were
  active.
- Expected listeners were present on TCP `8000`, `8080`, `8081`, and
  localhost-only `4317`.
- Local RPC returned block `78,726,044` for `eth_blockNumber`.
- Execution advanced through block `78,726,048` during the validation window.
- BFT logs showed committed blocks, successful votes, and vote sending around
  block `78,726,002` and later.
- No restart loop was observed.

Resource state after recovery:

- Memory remained acceptable, with roughly `31 GiB` available and no swap in
  use.
- Service memory shortly after recovery was roughly `3.8 GiB` for BFT,
  `3.4 GiB` for execution, `234 MiB` for RPC, and `10 MiB` for the local OTEL
  collector.
- CPU `Tctl` rose back into the normal sustained-load range for this mini PC,
  around `86.6 C`.

The CPU temperature was hot, but consistent with prior observed node load on
this underspecced mini PC. It should continue to be monitored for rapid climbs
toward shutdown territory, throttling, service instability, or sustained
performance drops.

## Notable Log Classification

During the recovery, RPC logged short triedb cache traversal errors for one
proposed block while starting. The node then continued to serve RPC and advance
blocks. Based on the follow-up checks, these lines were treated as recovery-time
cache/database traversal warnings rather than proof of ongoing node failure.

The earlier repeated blocksync timeout and "not available" messages were
node-impacting while RPC was unavailable. They resolved after the controlled
service restart and state-sync completion.

Desktop/session warnings from the boot were not treated as node-impacting
because the Monad services, listeners, memory, disk, and execution checks
provided more direct evidence of node health.

## Operator Takeaways

- After an abrupt power loss, active services alone are not enough to declare
  the node healthy. Confirm listeners, local RPC, execution progress, and BFT
  commit/vote activity.
- If RPC is stuck at `Waiting for statesync to complete`, give it a short
  recovery window and check whether execution and BFT are advancing.
- If the node remains active but non-serving, a controlled service restart can
  restore normal state-sync/RPC behavior without touching node data.
- Avoid public logs that include peer lists, raw identities, public IP
  addresses, hostnames, usernames, passwords, wallet-like values, or sensitive
  service arguments.
