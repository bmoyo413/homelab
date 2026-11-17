# Incident: host IP drift broke Prometheus targets

**Date:** 2026-07-12
**Severity:** Low (monitoring blind spot, no user-facing outage)
**Status:** Closed, permanent fix applied
**Author:** bmoyo413
**Revised:** 2026-11-16 (detection and alerting corrected against my notes from the day, status updated)

> Addresses and identifiers below are sanitized placeholders.

## Summary

A monitored host moved from `192.168.1.20` to `192.168.1.30` because it runs
on DHCP with no reservation. Prometheus kept scraping the old address, so its
targets for that host went down and its metrics stopped until the config was
reconciled.

## Impact

- Prometheus targets for the host reported down.
- No production service outage; the host itself stayed up.

## Timeline

- **T0** Host reboots, its DHCP lease changes, and the address drifts `.20 -> .30`.
- **T1** Prometheus targets for the host flip to `down`; scrapes for its
  node-exporter and cAdvisor start failing.
- **T2** The address change is noticed by hand during unrelated troubleshooting on
  the host; the Prometheus targets page shows its scrape jobs `down` against the
  old address.
- **T3** Downstream references reconciled to the new address (see below), targets
  came back up.
- **T4** DHCP reservation added on the router (later, before the first build was
  retired).

## Detection

Found by hand, not by an alert. The host had been rebooted and I was diagnosing an
unrelated complaint about it when I saw its address had changed. The Prometheus
targets page confirmed the rest: its scrape jobs for the host were `down` against
the old address.

A `TargetDown` rule (`up == 0` for 5 minutes) existed at the time. It is not what
surfaced the problem, and my notes do not record whether its notification was
delivered and missed or never arrived. That is a second gap, separate from the
address change: an alert rule that nobody can vouch for end to end.

## Root cause

The host is on DHCP with no reservation, so its lease is free to change on
renewal or reboot. Every reference to it was pinned to a literal IP, so a single
address change silently invalidated all of them.

## Resolution and recovery

Reconciled the downstream references to the new address:
- Prometheus scrape config updated, targets came back up.
- Dashboard host variable updated.
- DNS: non-issue (wildcard record, no per-host entry needed).

## The permanent fix (done)

A DHCP reservation on the router maps the host to a fixed address by MAC. Until
it was in place, everything would have drifted again on the next reboot.

Scraping by hostname was not an option here: internal DNS was a single wildcard
pointing at the reverse proxy, so the host had no record of its own.

## Lessons learned

- Infra hosts that other systems scrape should never be on unreserved DHCP.
- A hardcoded IP in a config is a hidden coupling, and the drift stays
  invisible until a scrape fails.
- An alert rule is not detection until a test notification has been seen to
  arrive.

## Follow-up actions

- [x] DHCP reservation on the router for the host.
- [x] Reconcile the config-management inventory entry.
- [x] Consider: reserve-or-static policy for every scraped host.
- [ ] Fire a test `TargetDown` and confirm the notification arrives.
