# Monad Node Experiment

Public notes and reports from running a small pruned Monad node on reduced
hardware.

This experiment is focused on a mini-PC setup intended for one operator or a
small trusted group, not large public infrastructure or full-history archive
service. The node is used for local RPC/WebSocket access, direct transaction
execution, lower-latency high-frequency trading workflows, and reducing some
dependence on shared third-party endpoints.

## Start Here

- [Small pruned node setup guide](SETUP.pdf)

## Reports

- PDF system reports are in [`reports/pdf`](reports/pdf).
- All public TXT report files are bundled into
  [`reports/text-archive/public-report-text-files.zip`](reports/text-archive/public-report-text-files.zip).
- Latest published snapshot:
  [`monad-mini-pc-system-report-2026-05-20_0738.pdf`](reports/pdf/monad-mini-pc-system-report-2026-05-20_0738.pdf).

## Notes

The reports are intended for public sharing and are generated from local system
telemetry and Monad service logs. Private identity material, keys, wallet-like
values, public IPs, peer lists, hostnames, usernames, and sensitive command
arguments are intentionally excluded or redacted.

## June 6 Statesync Recovery (Stale Forkpoint)

Following the June 3 power loss, the node spent several days stuck in state sync.
All four Monad services reported `active`, but HTTP RPC and WebSocket RPC stayed
closed and execution did not advance. The cause was a stale consensus checkpoint:
the node was holding a forkpoint and validator set from epoch `1578` while the
network had advanced to epoch `1591`, so state sync was chasing a target that
providers no longer retain and could never complete.

The fix was to refresh the forkpoint and validator set together to the current
epoch-`1591` pair from the official configuration bucket, reset the local state
database for a clean sync base, restore the execution event-ring directory, and
restart the service group. State sync then re-targeted to the live chain height
and completed in roughly 42 minutes. Afterward all services were active, the
expected listeners were present on node networking, HTTP RPC, WebSocket RPC, and
the localhost-only OTEL endpoint, local RPC returned block `79,531,353`, and
execution advanced normally with no certificate-verification errors.

The sanitized incident note is available here:

- [June 6, 2026 statesync recovery (stale forkpoint)](reports/incidents/2026-06-06-statesync-wedge-recovery.md)

## June 2 Power-Loss Recovery

An unexpected local power outage rebooted the mini PC on June 2, 2026. The PC
itself came back cleanly, but the Monad node initially remained active without
serving local RPC: node networking and the local OTEL endpoint were listening,
while HTTP RPC and WebSocket RPC were still closed and `monad-rpc` was waiting
for state sync.

A controlled Monad service restart restored normal operation after a short
state-sync window. Local RPC and WebSocket RPC reopened, `eth_blockNumber`
returned block `78,726,044`, execution advanced through block `78,726,048`, and
BFT logs again showed committed blocks and successful vote activity.

The sanitized incident note is available here:

- [June 2, 2026 power-loss recovery](reports/incidents/2026-06-02-power-loss-recovery.md)

## May 20 Health Check

The node recovered cleanly after a local power/network interruption earlier in
the week. During the outage recovery, the Monad services started but BFT had no
usable peers until the router was power-cycled and BFT was restarted. After
that, the node resumed normal peer activity, local RPC came back on both HTTP
and WebSocket ports, and block execution advanced again.

The May 20 public snapshot shows the node in a healthy operating state:

- `monad-bft`, `monad-execution`, `monad-rpc`, and the local OTEL collector were
  active.
- Expected listeners were present on node networking, HTTP RPC, WebSocket RPC,
  and the localhost-only OTEL endpoint.
- Local RPC advanced from block `75,838,850` to `75,838,871` during a short
  check window.
- Recent BFT logs showed committed blocks, successful votes, and vote sending.
- Recent execution logs showed active block execution around block `75,838,849`
  to `75,838,852`.

Thermals remained in the expected sustained-load range for this mini PC. The
manual check saw CPU `Tctl` around `85.0 C` to `86.8 C`, and the generated
public report recorded a maximum sensor reading of `86.4 C`. That is hot, but
reasonable for this underspecced small-node experiment under continuous Monad
load. The operator is watching for rapid climbs toward the high 90s, throttling,
service instability, or sudden performance drops rather than treating every
mid-80s reading as a failure.

Maintenance note: the OS currently reports that a reboot is required, and apt
shows a stable `monad 0.14.4` package available over the installed
`0.14.4~rc.1` package, along with Docker, browser, and Tailscale updates. Treat
that as planned maintenance: update and reboot only when node downtime is
acceptable, then verify services, listeners, execution activity, and local RPC
after startup.

## May 16 Version Check

After Keone Hon posted on X that Monad `v0.14.3` requires Authenticated UDP for
RaptorCast, the local install was checked to decide whether a version change was
needed. Authenticated UDP is relevant to this experiment because reducing
consensus-message verification overhead is useful on underspecced pruned-node
hardware.

The installed package was `monad 0.14.4~rc.1` from the Category Labs package
repository, with `monad-node` and `monad-rpc` reporting `v0.14.4-rc.1`.
Although `v0.14.3` was the latest confirmed stable release at the time of the
check, the installed release-candidate build was already newer. A local
non-sensitive binary and log check confirmed that the running BFT service
included the Authenticated UDP/RaptorCast code path and was using the
wire-authenticated transport path.

Decision: do not downgrade solely to reach `v0.14.3`. This node already has the
Authenticated UDP generation of networking changes, so the operator kept the
current install while continuing to watch for official Monad/Category Labs
operator guidance on whether to prefer the stable `v0.14.3` release over the
`v0.14.4` release candidate.

Report event sections classify scary-looking log lines so readers can
distinguish node-impacting issues from report collection timeouts, expected
pruned-node limitations, WebSocket/client disconnect noise, and historical
issues that were later resolved.

This is a pruned-node experiment. Historical data older than the node's local
retention window, roughly 20 days for this setup, should be backfilled from an
archive source, exchange, indexer, explorer, or another data platform as needed.
