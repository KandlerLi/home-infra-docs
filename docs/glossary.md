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
| home-agent | Deployed model-facing API container, reachable only on homeserver loopback port 8090 |
| home-tools | Deployed fixed read-only host information service using a protected Unix socket |
| Infrastructure repository | `/home/julian/projects/home-infra` |
| SOPS | Secrets OPerationS, used to encrypt Ansible variables in Git |
| Runner VM | Dedicated Debian virtual machine for self-hosted GitHub Actions jobs |
