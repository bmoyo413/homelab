# Incident: host IP drift broke Prometheus targets

**Date:** 2026-07-12
**Severity:** Low (monitoring blind spot, no user-facing outage)
**Status:** Mitigated, permanent fix pending
**Author:** bmoyo413

> Addresses and identifiers below are sanitized placeholders.

## Summary

A monitored host moved from `192.168.1.148` to `192.168.1.118` because it runs
on DHCP with no reservation. Prometheus kept scraping the old address, so its
targets for that host went down and metrics and alerts for it went blind until
the config was reconciled.

## Impact

- Prometheus targets for the host reported down.
- No production service outage; the host itself stayed up.

## Timeline

<!-- Absolute times. Fill from memory / logs. -->
- **T0** host reboots / lease changes, address drifts `.148 -> .118`.
- **T1** Prometheus targets for the host show down.
- **T2** Detected via ...
- **T3** Reconciled downstream references (see below).

## Detection

<!-- How you found out. Dashboard? Alert? Eyeball? Be specific: this section is
     the difference between "I got lucky" and "my monitoring works". -->

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
- A literal IP in a config is a hidden coupling; a drift is invisible until a
  scrape fails.

## Follow-up actions

- [ ] DHCP reservation on the router for the host.
- [ ] Reconcile the config-management inventory entry.
- [ ] Consider: reserve-or-static policy for every scraped host.
