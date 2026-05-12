# Monad Node Experiment

Public notes and reports from running a small pruned Monad node on reduced
hardware.

This experiment is focused on a mini-PC setup intended for one operator or a
small trusted group, not large public infrastructure or full-history archive
service. The node is used for local RPC/WebSocket access, direct transaction
execution, lower-latency bot workflows, and reducing some dependence on shared
third-party endpoints.

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

This is a pruned-node experiment. Historical data older than the node's local
retention window, roughly 20 days for this setup, should be backfilled from an
archive source, exchange, indexer, explorer, or another data platform as needed.
