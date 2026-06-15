# BIG-IP LTM Automation (Ansible)

Declarative management of Virtual Servers, Pools, Nodes and custom TCP/SSL
Profiles on F5 BIG-IP using the
[`f5networks.f5_modules`](https://galaxy.ansible.com/f5networks/f5_modules)
collection.

All profiles and services are described in a single file
([vars/services.yml](vars/services.yml)). Adding a new service means appending a
YAML block — no role or playbook changes.

The catalog ships two example services that mirror the guide: an unencrypted
HTTP path and an encrypted HTTPS (TLS) path that terminates client TLS and
re-encrypts to the Dell OBS nodes.

## Layout

```
ansible.cfg
requirements.yml / requirements.txt
inventories/default/         # hosts + group_vars
playbooks/
  site.yml                   # deploy everything
  deploy_service.yml         # deploy one service (-e service_name=...)
  teardown_service.yml       # remove (state=absent, reverse order)
  tasks/_filter_services.yml # shared service-name filter
roles/
  bigip_common/              # ENV check + connection smoke test
  bigip_ssl_certs/           # generate self-signed certs + bigip_ssl_key/certificate
  bigip_profiles/            # bigip_profile_tcp + bigip_profile_{client,server}_ssl
  bigip_nodes/               # bigip_node
  bigip_pool/                # bigip_pool + bigip_pool_member (connection limit)
  bigip_virtual_server/      # bigip_virtual_server
vars/services.yml            # profiles + service catalog (single source of truth)
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

## Profiles and service definition

[vars/services.yml](vars/services.yml) has two top-level keys: `profiles`
(custom TCP/SSL profiles created once and shared by services) and `services`
(the pools, virtual servers and nodes).

```yaml
profiles:
  tcp:
    - { name: s3-tcp-custom-client, parent: s3-tcp, nagle: auto, congestion_control: cubic }
    - { name: s3-tcp-custom-server, parent: f5-tcp-lan }
  client_ssl:
    - name: clientssl-dell-profile
      parent: clientssl
      cert: clientssl-dell           # cert/key object names created on BIG-IP
      key: clientssl-dell
      self_signed:                   # generate + upload a self-signed lab cert
        common_name: dell-s3.example.com
        days: 825
  server_ssl:
    - { name: serverssl-dell-profile, parent: serverssl }   # parent default = "Server Certificate: Ignore"

services:
  # Encrypted path: TLS client-side, re-encrypted to the OBS nodes (9021).
  - name: dell_s3_https_cluster
    tenant: Common                 # optional, default = Common
    virtual_server:
      destination: 100.1.1.182     # external VIP
      port: 443
      profiles:                    # a bare name = context 'all'; a mapping pins the side
        - { name: s3-tcp-custom-client, context: client-side }
        - { name: s3-tcp-custom-server, context: server-side }
        - http
        - { name: clientssl-dell-profile, context: client-side }
        - { name: serverssl-dell-profile, context: server-side }
      snat: automap
    pool:
      lb_method: least-connections-member
      monitor: https
      members:                     # connection_limit defaults to 1000 (guide 2.3.1)
        - { name: dell_node_01, address: 172.16.0.161, port: 9021 }
        - { name: dell_node_02, address: 172.16.0.162, port: 9021 }
```

Resulting BIG-IP object names:

- pool: `dell_s3_https_cluster_pool`
- virtual server: `dell_s3_https_cluster_vs`
- nodes: `dell_node_01`, `dell_node_02`, …
- profiles: `s3-tcp-custom-client`, `s3-tcp-custom-server`,
  `clientssl-dell-profile`, `serverssl-dell-profile`

> The HTTP and HTTPS example services share the same nodes — a BIG-IP node is
> keyed by address, so the names match and only the pool member port differs
> (9020 vs 9021).

### TLS certificates

When a `client_ssl` profile carries a `self_signed:` block, the
`bigip_ssl_certs` role generates a private key and self-signed certificate on
the control node (via `community.crypto`), writes them to a gitignored `.tls/`
directory, uploads them to BIG-IP (`bigip_ssl_key` / `bigip_ssl_certificate`)
under the `cert`/`key` names, and the client SSL profile then references them.
This requires the `cryptography` Python library (in `requirements.txt`).

The generated certificate is for lab use — S3 clients won't trust it, so the
WARP test in the guide uses `--insecure`. For production, **drop the
`self_signed:` block** and pre-import a CA-signed certificate and key under the
same `cert`/`key` object names (System > Certificate Management).

## Run

```bash
# dry-run first
ansible-playbook playbooks/site.yml --check --diff

# deploy all services
ansible-playbook playbooks/site.yml

# deploy a single service
ansible-playbook playbooks/deploy_service.yml -e service_name=dell_s3_https_cluster

# run a single stage
ansible-playbook playbooks/site.yml --tags ssl_certs
ansible-playbook playbooks/site.yml --tags profiles
ansible-playbook playbooks/site.yml --tags nodes
ansible-playbook playbooks/site.yml --tags pool
ansible-playbook playbooks/site.yml --tags vs

# remove a service (custom profiles are left in place — see teardown notes)
ansible-playbook playbooks/teardown_service.yml -e service_name=dell_s3_https_cluster
```

## Extending

| Need                              | Action                                                                       |
|-----------------------------------|------------------------------------------------------------------------------|
| New service                       | Append a block under `services:` in [vars/services.yml](vars/services.yml)   |
| New custom profile                | Append a block under `profiles:` in [vars/services.yml](vars/services.yml)   |
| Different tenant (partition)      | Set `tenant: MyTenant` on the service                                        |
| Change defaults (port, monitor, connection limit) | Edit `roles/<role>/defaults/main.yml`                       |
| Self-signed lab cert              | Add a `self_signed:` block to the client_ssl profile (auto-generated)        |
| Present a CA-signed cert          | Drop `self_signed:`; pre-import a cert/key under the `cert`/`key` names       |
| Multiple BIG-IPs                  | Add hosts to [inventories/default/hosts.yml](inventories/default/hosts.yml)  |
| Custom health monitor             | Add a `bigip_monitor` role (e.g. `bigip_monitor_http`) + service field       |
| HA / config-sync                  | Add a play with `bigip_configsync_action` after deploy                       |
| iRules                            | Add a `bigip_irule` role                                                     |

## Assumptions

- VLANs, Self-IPs and base networking on the BIG-IP are already configured.
- Custom TCP/SSL profiles are created by the `bigip_profiles` role, but
  `bigip_profile_tcp` does not expose every knob the guide tunes in the GUI
  (minimum RTO, proxy buffer high/low, auto receive window, tail loss). Those
  are inherited from the parent profile — apply the remaining fine-tuning from
  guide section 2.3.2 manually or via tmsh if your workload needs it.
- The client SSL profile presents a self-signed certificate generated by the
  `bigip_ssl_certs` role (the `self_signed:` block). This needs the
  `cryptography` library on the control node and writes private keys to a
  gitignored `.tls/` directory. For production, drop `self_signed:` and import a
  CA-signed cert/key under the same names. See [TLS certificates](#tls-certificates).
- The server SSL profile inherits the parent `serverssl` defaults, which do not
  validate the backend certificate ("Server Certificate: Ignore"). Front a
  trusted CA before using this in production.

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
tmsh list ltm virtual dell_s3_https_cluster_vs
tmsh list ltm pool    dell_s3_https_cluster_pool
tmsh list ltm node
tmsh list ltm profile tcp s3-tcp-custom-client
tmsh list ltm profile client-ssl clientssl-dell-profile
tmsh list ltm profile server-ssl serverssl-dell-profile
```
