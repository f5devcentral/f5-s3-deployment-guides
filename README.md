# F5 S3 Deployment Guides

Reference deployment guides for placing an **F5 BIG-IP Local Traffic Manager (LTM)** in front of an S3-compatible object storage cluster. Each guide walks through the network, pool, and virtual server configurations needed to expose a multi-node storage cluster behind a single, highly available S3 endpoint — turning a set of individual storage nodes into a single resilient, load-balanced service.

The guides share a common architecture: S3 clients connect to a BIG-IP virtual server (the single S3 endpoint), and the BIG-IP load balances requests across the storage cluster nodes, monitoring node health and applying TCP and TLS optimizations tuned for S3 traffic. Each guide is self-contained and includes an optional Ansible path that reproduces the manual steps declaratively.

> **Note:** These documents are demonstration content intended to illustrate reference deployments. Substitute your own addresses, naming conventions, security policies, and operational practices before applying any of these steps in a production environment.

## Guides

| Storage platform | Guide | Automation |
| ---------------- | ----- | ---------- |
| **Dell ObjectScale** | [Dell/README.md](./Dell/README.md) | [Dell/automation/](./Dell/automation/) |
| **MinIO AIStor** | [MinIO/README.md](./MinIO/README.md) | [MinIO/automation/](./MinIO/automation/) |
| **NetApp StorageGRID** | [NetApp/README.md](./NetApp/README.md) | [NetApp/automation/](./NetApp/automation/) |

Each guide covers:

- **Setup** — lab topology, component versions, and the network addressing used in the examples.
- **BIG-IP configuration** — VLANs, Self IPs and routes, health monitors, the backend pool, and the client-facing virtual server.
- **S3 traffic optimization** — custom client- and server-side TCP profiles, node connection limits, and optional client/server SSL profiles for TLS termination and re-encryption.
- **Validation** — configuring a bucket, user, and access key on the storage cluster, then driving a mixed S3 workload through the virtual server with the [MinIO WARP](https://www.min.io/download/minio-warp) benchmarking tool to confirm end-to-end connectivity.
- **Automation with Ansible** — a declarative alternative to the GUI walkthrough.

## Prerequisites

- An F5 BIG-IP running a supported version (the guides use BIG-IP 21.1.0; the Dell guide also covers rSeries/F5OS tenant provisioning).
- A deployed, reachable S3 object storage cluster (Dell ObjectScale, MinIO AIStor, or NetApp StorageGRID).
- Administrative access to both the BIG-IP and the storage cluster.
- Familiarity with basic L2/L3 networking concepts.
- For validation: the [MinIO WARP](https://www.min.io/download/minio-warp) CLI and `jq`.
- For the automation path: Python 3, Ansible, and the [`f5networks.f5_modules`](https://galaxy.ansible.com/f5networks/f5_modules) collection (see each guide's `automation/README.md`).

## Getting started

1. Pick the guide that matches your storage platform from the table above.
2. Follow the setup section to confirm your topology and addressing.
3. Work through the BIG-IP configuration, adapting the example addresses and names to your environment.
4. Apply the S3 traffic optimizations appropriate to your workload.
5. Validate the deployment with a WARP test through the virtual server.
6. Optionally, reproduce the configuration declaratively using the Ansible automation in the guide's `automation/` directory.

Each platform directory contains a `README.md` (the guide), an `assets/` directory of screenshots, and an `automation/` directory with Ansible playbooks, roles, and inventory.

## License

See the repository's license and policy files for terms, contribution guidelines, and the security policy.
