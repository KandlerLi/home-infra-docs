# Runbook: Operate Nextcloud AIO

## Current model

Ansible manages the AIO master container. The master container manages the
Nextcloud, Apache, PostgreSQL, Redis, Collabora, Talk, Imaginary, Notify Push,
and Whiteboard containers. A separate Ansible-managed Traefik container owns
public TCP ports 80/443 and routes `nextcloud.jkandler.de` to AIO Apache on
`127.0.0.1:11000`.

## Read-only health check

```bash
ssh julian@192.168.178.100
docker ps
df -hT / /mnt/black-hdd
systemctl --failed
```

Expected baseline:

- all ten AIO containers are running
- every observed AIO health status is healthy
- `/mnt/black-hdd` is mounted
- Traefik owns TCP ports 80/443 and is healthy
- AIO Apache publishes only `127.0.0.1:11000`
- AIO management ports 8080/8443 and Talk port 3478 match the architecture

## Before an Ansible change

1. Verify the HDD mount.
2. Review changes to image, ports, volumes, and `NEXTCLOUD_DATADIR`.
3. Confirm the AIO configuration volume name is unchanged.
4. Take or verify an appropriate backup.
5. Ensure no unrelated task changes ownership below `/mnt/black-hdd`.

## Never perform as routine cleanup

- delete `nextcloud_aio_*` volumes
- recreate the master configuration volume
- move the data directory
- expose the Docker socket to another application
- add an AAAA record based only on the presence of a host IPv6 address
- change ownership of ports 80/443 outside the shared-ingress runbook

## Reverse-proxy operation

The AIO instance runs in its supported reverse-proxy mode. The desired state is
persisted in Ansible with `nextcloud_aio_reverse_proxy_enabled`,
`shared_ingress_enabled`, and `shared_ingress_start` set to true. Do not turn
these off during an ordinary apply. See ADR 0005 and the publish-home-agent
runbook for the guarded rollback procedure.

## Shut down for physical hardware maintenance

Use the tracked helper instead of cutting power or manually stopping individual
containers. The helper checks that `/mnt/black-hdd` is mounted, refuses to
continue while a libvirt VM is running, asks AIO to stop its application
containers, verifies the stop, flushes filesystem buffers, and requests an OS
poweroff. It does not create a backup.

From the controller:

```bash
cd /home/julian/projects/infra/home-infra

scp scripts/shutdown-homeserver-for-maintenance.sh \
  julian@192.168.178.100:/tmp/

ssh -t julian@192.168.178.100 \
  'sudo bash /tmp/shutdown-homeserver-for-maintenance.sh --check'
```

Verify that users have finished active uploads and that an appropriate recent
backup exists. Then start the guarded shutdown and type the exact confirmation
`POWER OFF`:

```bash
ssh -t julian@192.168.178.100 \
  'sudo bash /tmp/shutdown-homeserver-for-maintenance.sh'
```

If the script reports an error after beginning the AIO stop, do not cut power.
Follow its warning, inspect the remaining containers, and either rerun the
script after resolving the problem or start the application containers again
through the AIO interface.

After the chassis is fully off, disconnect mains power and discharge standby
power before opening it. Following the hardware work, boot the host and verify:

```bash
free -h
systemctl --failed
docker ps
curl --fail-with-body https://nextcloud.jkandler.de/status.php
```

The 2026-08-19 RAM maintenance confirmed that the deliberately stopped AIO
application containers remained stopped after the host boot. Start them through
the AIO interface at `https://192.168.178.100:8080`, then repeat the checks.
