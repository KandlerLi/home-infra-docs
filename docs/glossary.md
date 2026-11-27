# Glossary

| Term | Meaning in this environment |
| --- | --- |
| AIO | Nextcloud All-in-One, the master container and its managed sibling containers |
| C4 | Architecture model using software system, container, component, and code abstractions |
| Container (C4) | A deployable/runnable application or data store; not necessarily a Docker container |
| Container (Docker) | A process isolated and managed by Docker Engine |
| Controller | The machine from which Ansible is executed |
| DDNS | Dynamic DNS, used to track a changing public IPv4 address |
| GPG | GNU Privacy Guard, used as the SOPS key mechanism |
| HDD | The large ext4 data disk mounted at `/mnt/black-hdd` |
| Home Infrastructure | The complete managed homeserver environment and supporting automation |
| Infrastructure repository | [`home-infra`](https://github.com/KandlerLi/home-infra) |
| SOPS | Secrets OPerationS, used to encrypt Ansible variables in Git |
| Runner | GitHub Actions job host: GitHub-hosted (`ubuntu-latest`) for most repos, an ARC runner Pod on `k3s-node-2` for repos whose CI needs the home network (today `stalwart-config`); the runner VM was retired 2026-08-31 |
