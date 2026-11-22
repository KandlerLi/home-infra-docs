# ADR 0036: Minecraft server on k3s, public on 25565/tcp and 19132/udp+tcp

- Status: Proposed (2026-11-22)
- Date: 2026-11-22
- Deployment status: live since 2026-11-22 (Java); Bedrock needs the NetherNet amendment below

## Context

Julian wants a Minecraft server for up to 8 players, Java and Bedrock,
reachable from the internet. The game protocols are raw TCP (Java) and
UDP (Bedrock), so Traefik and Authelia cannot sit in front of them. That
means new raw ports in `k3s_ingress_forward`, a list the security
invariants keep closed (80, 443, 53, 25, 993, 465): adding one is a
deliberate decision, which this ADR records.

Options considered:

- **Paper with Geyser and Floodgate** (`itzg/minecraft-server`): one Java
  server; Geyser translates the Bedrock protocol, Floodgate lets Bedrock
  players in without a Java account. One world, one Pod, one backup. Java
  game rules apply to everyone, and Bedrock-only content (Marketplace,
  add-ons) is unavailable.
- **Paper plus a separate Bedrock Dedicated Server**: two worlds, two
  Pods, two backups, and Java and Bedrock players can't meet.
- **LAN only**: no new public ports, but friends outside the home
  network can't play.

## Decision

Paper with Geyser and Floodgate in `apps`, as `k3s-apps/argocd/minecraft`,
public on 25565/tcp (Java) and 19132/udp (Bedrock).

- **Exposure**: the FRITZ!Box forwards both ports to the homeserver
  (`fritzbox`), `k3s_ingress_forward` DNATs them to `minecraft-svc`
  (`externalTrafficPolicy: Local`, so the server sees client IPs). RCON
  (25575) stays inside the Pod: not in the Service, not forwarded.
- **Access control is the game's own**: `online-mode` (Mojang/Xbox
  account verification) and an enforced whitelist, which starts empty.
  Players are added to `WHITELIST` in the Deployment: Java profile names,
  Bedrock players by their Floodgate UUID.
- **NetworkPolicy**: in only on the two game ports; out only DNS and
  HTTPS to public addresses (Mojang session servers, Paper and plugin
  downloads), never the LAN or the cluster.
- **Rootless** as 980:975 under PodSecurity `restricted`, the ids of the
  homeserver's `minecraft-backup` account, which owns both the world
  directory on `k3s-node-1` and the backup directory.
- **RCON password**: generated per Pod start by an init container into a
  memory `emptyDir`, shared only with the backup sidecar. It never leaves
  the Pod, so nothing goes into Secrets Manager.
- **Pinned** image digests and Minecraft version (26.3); Geyser and
  Floodgate track their latest build, because Bedrock clients update
  themselves and only the latest Geyser speaks their protocol.
- **Backup**: `itzg/mc-backup` sidecar, every 24 hours, `save-off` over
  RCON, tar to `/mnt/red-hdd/minecraft-backup` over NFS, 14 days kept.
  `backup-freshness` watches the newest tarball.
- **Sizing**: 3 GiB heap, 4 GiB memory limit; `k3s-node-1` grows from 10
  to 14 GiB. No CPU limit, like every workload here; view distance 8 and
  simulation distance 6 for the i7-2600's single-core speed.

## Amendment (2026-11-22): NetherNet for Bedrock

Found on the first start: Bedrock 26.60 (2026-10-27) no longer connects
over RakNet, only over NetherNet (WebRTC), and Geyser 2.12 defaults to it.
NetherNet needs signaling, which Geyser's built-in mode serves over TCP on
the Bedrock port, and it tells players which address to connect to.

- **19132/tcp is forwarded too**, in `fritzbox`, `k3s_ingress_forward`,
  `minecraft-svc` and the NetworkPolicy.
- **`minecraft.jkandler.de`**, a CNAME to the DynDNS apex (`dyndns`), is
  the advertised address; STUN discovery is off, since the NetworkPolicy
  blocks it and the address is known.
- **`auth-type: floodgate`**: Geyser does not switch on its own, so Bedrock
  players needed a Java account before.
- Geyser's config takes no env vars; a ConfigMap (`minecraft-geyser-patches`)
  holds `PATCH_DEFINITIONS` that the image applies on every start.
- Signaling stays plain HTTP: each Bedrock player confirms a "trust on
  first use" prompt once. HTTPS would need a certificate for
  `minecraft.jkandler.de` in the Pod.
- Geyser resolves the advertised host name when signaling starts. Whether
  it follows a changed home IP without a restart is untested.

## Consequences

- A second public, unauthenticated-at-the-edge service besides mail.
  Its attack surface is the Paper server and two plugins; a compromise is
  contained to the Pod (rootless, no service account token, egress only to
  public HTTPS).
- The homeserver's public IP becomes visible to everyone who plays, and
  to scanners that probe 25565.
- About 4 GiB more memory on `k3s-node-1`; the VM resize restarts it once.
- Geyser can lag a few days behind a new Bedrock release; until then,
  Bedrock players on the newest app version can't join.
- Rollback: remove the two `fritzbox` forwards and the two
  `k3s_ingress_forward` rules; the world stays on its `Retain` volume.
