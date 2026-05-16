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

## Notes

The reports are intended for public sharing and are generated from local system
telemetry and Monad service logs. Private identity material, keys, wallet-like
values, public IPs, peer lists, hostnames, usernames, and sensitive command
arguments are intentionally excluded or redacted.

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
