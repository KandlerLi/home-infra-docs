# Known issue: e1000e NIC watchdog-hang bug (root cause unconfirmed)

Status: **Root cause not confirmed.** Two `ethtool` offload/EEE
settings are disabled and a systemd watchdog-recovery safety net is
applied and believed to help, but the mitigation is not proven to hold
-- a recurrence should now turn into a fast, automatic PCI-level reset
instead of the original 14-minute-to-9.5-hour range of full outages.
Not something anyone is actively investigating further beyond watching
for whether it recurs -- kept here as a known issue, not an open task.

## What happened

**First occurrence, 2026-09-01 (~19:48-20:02 CEST, ~14 min)**: the
homeserver went completely unreachable over the network (ARP itself
stopped getting replies) while still powered on. Root cause: a
well-documented Intel e1000e/82579LM PCIe ASPM bug -- `NETDEV WATCHDOG
... transmit queue timed out` / `Reset adapter unexpectedly` looping
every ~7s, network fully dead during each cycle. Self-recovered after
~14 minutes.

**Attempted fix, 2026-09-04** (`infra/home-infra`, `ansible/roles/base`):
`pcie_aspm=off` kernel boot parameter. First signal looked clean.

**Recurred, 2026-09-06** (~9.5h outage, 01:13-09:07 CEST):
`pcie_aspm=off` confirmed active in `/proc/cmdline` throughout -- did
not prevent the recurrence. This time the driver's own internal reset
never recovered on its own -- `Detected Hardware Unit Hang` repeated
roughly every 2s continuously until a manual reboot, worse than the
first incident's self-recovery.

**Second fix, 2026-09-06** (`infra/home-infra`, `ansible/roles/base`):
the other two contributors most commonly cited alongside ASPM for this
exact chipset's bug -- TSO/GSO/GRO offloading and Energy Efficient
Ethernet -- both disabled via `ethtool` (applied immediately, persisted
via an `ifupdown` post-up hook). Plus a safety net regardless of
whether that actually eliminates the trigger:
`e1000e-watchdog-recovery.service`, a small systemd unit watching the
kernel log for a `NETDEV WATCHDOG` event and forcing a full PCI-level
unbind/rebind of the device the moment one fires, rather than trusting
the driver's own internal reset to recover -- turns a recurrence into
seconds of disruption instead of an all-night outage.

Neither fix has been proven by sustained recurrence-free uptime alone
(the same intermittent-bug caveat applied to the first attempt) --
real confirmation is the safety net actually being watched and, over
time, no further multi-hour outages.

## How this differs from the homeserver silent-freeze issue

See `homeserver-silent-freeze.md`'s own "How this differs" section for
the full comparison. Short version: the silent freeze's kernel logged
*nothing* during its dead window and `e1000e-watchdog-recovery.service`
never fired for it, whereas this bug's kernel stays alive and actively
logging (`NETDEV WATCHDOG` / `Detected Hardware Unit Hang`) throughout
its episodes. Two separately-tracked issues with different signatures,
not the same fault.

## If it recurs

1. Check `journalctl -t e1000e-watchdog-recovery` for whether the
   recovery service actually fired, and `journalctl -k` for whether
   `NETDEV WATCHDOG` / `Detected Hardware Unit Hang` still occurs at
   all.
2. If the recovery service fired and the network came back within
   seconds: the safety net worked as designed. Root cause is still
   unknown; this runbook doesn't change, just note the recurrence.
3. If the hang happens but the recovery service does *not* fire, or
   the network doesn't recover even after a PCI unbind/rebind:
   escalate -- the mitigation isn't holding, and it's worth
   reconsidering the `ethtool` offload changes or a driver/firmware
   update, not just restating the same fix.

- Repo: `infra/home-infra` (`ansible/roles/base`)
