# BIG-IP LTM Automation (Ansible)

Declarative management of Virtual Servers, Pools and Nodes on F5 BIG-IP using
the [`f5networks.f5_modules`](https://galaxy.ansible.com/f5networks/f5_modules)
collection.

All services are described in a single file ([vars/services.yml](vars/services.yml)).
Adding a new service means appending a YAML block — no role or playbook changes.

## Layout

```
ansible.cfg
requirements.yml / requirements.txt
inventories/default/         # hosts + group_vars
playbooks/
  site.yml                   # deploy everything
  deploy_service.yml         # deploy one service (-e service_name=...)
  teardown_service.yml       # remove service and monitor (state=absent, reverse order)
  tasks/_filter_services.yml # shared service-name filter
roles/
  bigip_common/              # ENV check + connection smoke test
  bigip_monitor_http/        # custom MinIO AIStor write quorum HTTP monitor
  bigip_nodes/               # bigip_node
  bigip_pool/                # bigip_pool + bigip_pool_member
  bigip_virtual_server/      # bigip_virtual_server
vars/services.yml            # service catalog (single source of truth)
```

## Install

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml

# (optional) dev tooling: linters
pip install -r requirements-dev.txt
```

## Credentials

F5 modules read connection settings from environment variables:

```bash
export F5_HOST=bigip01.example.com
export F5_USER=admin
export F5_PASSWORD=...
export F5_SERVER_PORT=443         # optional, default 443
export F5_VALIDATE_CERTS=false    # optional, default false
```

Apply variables

```bash
source .env
```

No Ansible Vault is used.

## Service definition

[vars/services.yml](vars/services.yml):

```yaml
services:
  - name: minio_s3_http_cluster
    tenant: Common              # optional, default = Common
    virtual_server:
      destination: 10.1.40.151  # external VIP
      port: 9000
      profiles: [http, tcp]
      snat: automap
    pool:
      lb_method: round-robin
      monitor: minio-health-check
      members:
        - { name: minio_node_http_29, address: 10.1.10.100, port: 9000 }
        - { name: minio_node_http_30, address: 10.1.10.101, port: 9000 }
```

Resulting BIG-IP object names:

- pool: `minio_s3_http_cluster_pool`
- virtual server: `minio_s3_http_cluster_vs`
- nodes: `minio_node_http_29`, `minio_node_http_30`

## Run

```bash
# dry-run first
ansible-playbook playbooks/site.yml --check --diff

# deploy all services
ansible-playbook playbooks/site.yml

# deploy a single service
ansible-playbook playbooks/deploy_service.yml -e service_name=minio_s3_http_cluster

# run a single stage
ansible-playbook playbooks/site.yml --tags monitor
ansible-playbook playbooks/site.yml --tags nodes
ansible-playbook playbooks/site.yml --tags pool
ansible-playbook playbooks/site.yml --tags vs

# remove the service and custom monitor
ansible-playbook playbooks/teardown_service.yml -e service_name=minio_s3_http_cluster
```

## Extending

| Need                              | Action                                                                       |
|-----------------------------------|------------------------------------------------------------------------------|
| New service                       | Append a block to [vars/services.yml](vars/services.yml)                     |
| Different tenant (partition)      | Set `tenant: MyTenant` on the service                                        |
| Change defaults (port, monitor)   | Edit `roles/<role>/defaults/main.yml`                                        |
| Multiple BIG-IPs                  | Add hosts to [inventories/default/hosts.yml](inventories/default/hosts.yml)  |
| Custom health monitor             | Edit `roles/bigip_monitor_http/defaults/main.yml` and set `pool.monitor`     |
| HA / config-sync                  | Add a play with `bigip_configsync_action` after deploy                       |
| iRules / SSL profiles as objects  | Add `bigip_irule` / `bigip_profile_client_ssl` roles                         |

## Assumptions

- VLANs, Self-IPs and base networking on the BIG-IP are already configured.
- Built-in profiles (`http`, `tcp`, `clientssl`) are used.
- The `minio-health-check` custom HTTP monitor is created before pools and removed after pool teardown.

## Lint / validation

Local:

```bash
yamllint --strict .
ansible-lint
ansible-playbook playbooks/site.yml --syntax-check
```

CI runs the same checks on every push / PR — see [.github/workflows/lint.yml](../.github/workflows/lint.yml).

## Idempotency check

```bash
ansible-playbook playbooks/site.yml   # first run:  changed > 0
ansible-playbook playbooks/site.yml   # second run: changed = 0
```

Verify on the BIG-IP:

```
tmsh list ltm virtual minio_s3_http_cluster_vs
tmsh list ltm pool    minio_s3_http_cluster_pool
tmsh list ltm node
```
