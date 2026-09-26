# Runbook: Admin VPN via the FRITZ!Box's WireGuard

Decision: [ADR 0025](../adr/0025-fritzbox-wireguard-admin-vpn.md).
This is a manual FRITZ!Box setting. There is no IaC path for it, so
this runbook is the record of how it is set up.

## Add a device

1. FRITZ!Box UI (`http://192.168.178.1`) → Internet → Freigaben →
   VPN (WireGuard) → **WireGuard-Verbindung hinzufügen**.
2. Choose the simplified setup for a **single device**, give it the
   device's name (e.g. `iphone-julian`), and finish. The FRITZ!Box may
   ask for an extra confirmation (a button on the box or a phone).
3. Import the result into the WireGuard app: QR code on the phone,
   the downloaded `.conf` file on a computer.
4. Adjust the client config in the app before first use:
   - `DNS = 192.168.178.100` (Blocky) instead of the FRITZ!Box.
   - `AllowedIPs`: keep `0.0.0.0/0` to send all traffic home (useful on
     untrusted Wi-Fi), or `192.168.178.0/24` for a split tunnel with
     only home traffic.
   - `Endpoint` can stay as generated, or use `vpn.jkandler.de` with
     the same port.

## Verify

From mobile data (not the home Wi-Fi), with the tunnel on:

```bash
ssh julian@192.168.178.100
```

In the FRITZ!Box UI, the connection shows a recent "Letzte Aushandlung".

## Reach the k3s network

`192.168.101.0/24` isn't routed from the LAN, so it isn't from the VPN
either. Go through the homeserver, e.g. `ssh -L 6443:192.168.101.10:6443
julian@192.168.178.100` for the k3s API.

## Remove a device

Delete its entry in the same list. That revokes its key immediately.

## Recreate after a FRITZ!Box reset

Restore the FRITZ!Box settings export if one exists. Otherwise, add
each device again as above and re-import it on the device, because
the old keys are gone.
