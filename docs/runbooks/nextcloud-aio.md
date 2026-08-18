# Runbook: Operate Nextcloud AIO

## Current model

Ansible manages the AIO master container. The master container manages the
Nextcloud, Apache, PostgreSQL, Redis, Collabora, Talk, Imaginary, Notify Push,
and Whiteboard containers.

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
- ports 80, 8080, 8443, 443, and 3478 match the documented architecture

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
- place another service on port 443 without a planned ingress migration

## Reverse-proxy migration

The current AIO instance directly owns port 443. A future shared proxy must use
the officially supported AIO reverse-proxy mode, preserve the existing domain,
and include a tested rollback. See ADR 0005 and the proposed public agent
ingress diagram.
