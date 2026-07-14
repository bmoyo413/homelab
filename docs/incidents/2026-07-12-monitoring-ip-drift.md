# Incident: host IP drift broke Prometheus targets

**Date:** 2026-07-12
**Severity:** Low (monitoring blind spot, no user-facing outage)
**Status:** Mitigated, permanent fix pending
**Author:** bmoyo413

> Addresses and identifiers below are sanitized placeholders.

## Summary

A monitored host moved from `192.168.1.20` to `192.168.1.30` because it runs
on DHCP with no reservation. Prometheus kept scraping the old address, so its
targets for that host went down and metrics and alerts for it went blind until
the config was reconciled.

## Impact

- Prometheus targets for the host reported down.
- No production service outage; the host itself stayed up.

## Timeline

- **T0** Host reboots, its DHCP lease changes, and the address drifts `.20 -> .30`.
- **T1** Prometheus targets for the host flip to `down`; scrapes for its
  node-exporter and cAdvisor start failing.
- **T2** The gap surfaces on the Grafana host dashboard (no fresh series) and via
  the `up == 0` panel on that host.
- **T3** Downstream references reconciled to the new address (see below), targets
  came back up.

## Detection

Caught on the monitoring side rather than by an alert page: the host's panels on
the Grafana node dashboard stopped updating, and the Prometheus targets page
showed its scrape jobs `down` with a connection-refused error against the old
address. Because the outage was a monitoring blind spot and not a service
failure, nothing else surfaced it, which is exactly why the DHCP-reservation
follow-up below matters.

## Root cause

The host is on DHCP with no reservation, so its lease is free to change on
renewal or reboot. Every reference to it was pinned to a literal IP, so a single
address change silently invalidated all of them.

## Resolution and recovery

Reconciled the downstream references to the new address:
- Prometheus scrape config updated, targets came back up.
- Dashboard host variable updated.
- DNS: non-issue (wildcard record, no per-host entry needed).

## The permanent fix (pending)

Add a DHCP reservation on the router mapping the host to a fixed address by MAC.
Without the reservation, everything re-drifts on the next reboot.

## Lessons learned

- Infra hosts that other systems scrape should never be on unreserved DHCP.
- A hardcoded IP in a config is a hidden coupling, and the drift stays
  invisible until a scrape fails.

## Follow-up actions

- [ ] DHCP reservation on the router for the host.
- [ ] Reconcile the config-management inventory entry.
- [ ] Consider: reserve-or-static policy for every scraped host.
