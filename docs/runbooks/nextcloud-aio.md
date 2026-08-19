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
