# Known issue: homeserver total, silent freeze (root cause unknown)

Status: **Root cause not identified. A hardware watchdog backstop is
applied and confirmed live** — a recurrence becomes an automatic
reboot within seconds instead of a manually-discovered multi-hour
blackout. This bounds the damage; it does not fix the actual cause.
Not something anyone is actively investigating further — kept here as
a known issue in case it recurs, not as an open task.

## What happened

Real production outage 2026-09-11/12: an AWS health-check alert fired
at 23:04; the user rebooted the homeserver the next morning at ~07:51.
The box stayed powered on the whole time (fans running, lights on) but
was fully unresponsive — no SSH, no services.

Confirmed via the actual privileged journal, not assumed:

- `journalctl --list-boots` (unprivileged) misleadingly showed the
  boot's own "LAST ENTRY" as 22:39:49 — that's just the last entry
  visible to a non-`adm`/`systemd-journal` user, not the real end of
  the journal. The real, privileged `journalctl -b -1` continues well
  past that (routine `dhcpcd` IPv6 route churn at 22:41 and 22:48, a
  DHCP lease renewal for `k3s-node-1` at 22:52:05) and only truly ends
  at **22:58:14** — a `dhcpcd` line deleting an IPv6 default route on
  `enp0s25`, with no corresponding "adding route" ever logged
  afterward, unlike the two prior cycles.
- `journalctl -k -b -1 --since '2026-09-11 22:55:00'` returned **no
  entries at all** — zero kernel-level messages of any kind in the
  entire dead window. No `NETDEV WATCHDOG`, no panic, no OOM, no
  hardware error, and critically, no graceful-shutdown sequence either
  (ruling out a normal `sudo reboot` from a still-responsive shell).
- A full `ansible-playbook site.yml` run (triggered by rotating the
  Nextcloud OIDC pair, unrelated infra work) finished at 22:38:50 —
  ~20 minutes before the freeze started. Initially suspected as the
  trigger; the ~20 minute gap plus multiple intervening successful
  `dhcpcd` cycles weakens that correlation, but it's not ruled out
  either. Not confirmed either way.

## How this differs from the e1000e NIC watchdog-hang bug

This is a materially different signature from the separately-tracked
e1000e NIC watchdog-hang bug (`homeserver-e1000e-nic-watchdog-hang.md`):
that bug's kernel stayed
alive enough to log `Detected Hardware Unit Hang` continuously for
hours during its own 2026-09-06 recurrence. This freeze's kernel
logged nothing at all after 22:58:14 — a harder, more total freeze
(the whole kernel stopped scheduling, not one hardware component with
everything else still ticking). `e1000e-watchdog-recovery.service`'s
own log confirms it never fired for this incident.

## What was fixed: a hardware watchdog backstop, not the root cause

Applied 2026-09-12 (`infra/home-infra`, `ansible/roles/base`):
confirmed live this hardware has a working watchdog timer device
(`/dev/watchdog`, `/dev/watchdog0`/`1`) that systemd wasn't using
(`RuntimeWatchdogUSec=0`). Enabled via `RuntimeWatchdogSec=30s` in
`/etc/systemd/system.conf` (applied via `systemctl daemon-reexec`, not
just staged for next boot) — root-cause-agnostic: if the kernel ever
fully freezes again for any reason, the hardware forces a real reset
with no software involved.

Applied live 2026-09-12 via `site.yml --ask-become-pass` and confirmed
for real, not just assumed from a clean apply:

```bash
ssh julian@192.168.178.100 systemctl show -p RuntimeWatchdogUSec
# expect: RuntimeWatchdogUSec=30s
```

## If it recurs

1. Check whether the watchdog actually fired:
   `journalctl -k | grep -i watchdog` around the recovery boot, or
   simply confirm the reboot was automatic vs. another manual
   power-cycle was needed.
2. If the watchdog **did** fire (automatic reboot, no manual
   intervention needed): the backstop worked as designed. Root cause
   is still unknown; this runbook doesn't change, just note the
   recurrence.
3. If the watchdog **did not** fire and another manual power-cycle was
   needed: the freeze is severe enough to also stop the watchdog's own
   hardware timer circuit (rare) — this would point toward a genuine
   hardware fault (PSU, motherboard, or RAM) rather than "just" a
   kernel software lockup, and is worth escalating into real
   investigation rather than treating as a repeat of the same known
   issue.

See the `investigate-homeserver-outage` skill for the general
journalctl/docker diagnostic sequence to run on any homeserver outage
report, independent of this specific known issue.
