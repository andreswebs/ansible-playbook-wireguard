# wireguard

Multi-cloud Ansible playbook for deploying a WireGuard server via `linuxserver/wireguard` and `docker compose`. Targets AWS, Azure, GCP, and Hetzner using a single base playbook and per-cloud connection shims.

## Layout

```txt
playbook.yml          # base: SSH path — used for GCP and Hetzner
playbook.aws.yml      # AWS: ansible_connection=aws_ssm
playbook.azure.yml    # Azure: az ssh / AAD login
inventory/
  aws_ec2.yml         # AWS EC2 dynamic inventory
  azure_rm.yml        # Azure dynamic inventory
  example.gcp.yml     # GCP dynamic inventory — copy to gcp_compute.yml and edit
  hcloud.yml          # Hetzner Cloud dynamic inventory
files/
  compose.yaml        # WireGuard service definition
  .env                # per-deployment env (SERVERURL, PEERS, ALLOWEDIPS)
```

All inventories produce the host group `vm_tag_wireguard` (Hetzner uses `app_wireguard` instead; the playbook's `hosts:` line accepts both via the union pattern).

## Prerequisites

### Controller-side

```sh
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml
```

The `azure.azcollection` collection requires extra Python packages — install after `ansible-galaxy collection install`:

```sh
pip install -r ~/.ansible/collections/ansible_collections/azure/azcollection/requirements.txt
```

Per-cloud CLI/auth setup expected on the controller:

- **AWS** — `aws configure sso` (or static creds) and an active SSO session.
- **Azure** — `az login`; the inventory plugin reuses the operator's `az login` context. No service principal required.
- **GCP** — `gcloud auth application-default login` (ADC).
- **Hetzner** — `export HCLOUD_TOKEN=...`.

### Target-VM-side

VMs must be tagged so the dynamic inventory groups them correctly:

| Cloud   | Tag / label                        | Resulting group                      |
| ------- | ---------------------------------- | ------------------------------------ |
| AWS     | `Name=wireguard`                   | `vm_tag_wireguard`                   |
| Azure   | `Name=wireguard`, `role=wireguard` | `vm_tag_wireguard`, `role_wireguard` |
| GCP     | network tag `wireguard`            | `vm_tag_wireguard`                   |
| Hetzner | label `app=wireguard`              | `app_wireguard`                      |

For Azure-specific VM prerequisites (AAD SSH extension, RBAC, NSG rules), see [docs/azure-prereqs.md](docs/azure-prereqs.md).

## Usage

### Hetzner / GCP — base playbook over plain SSH

```sh
./playbook.yml -i inventory/hcloud.yml
./playbook.yml -i inventory/gcp_compute.yml
```

### AWS — SSM connection

```sh
export AWS_REGION="eu-west-1"
export X_AWS_SSM_BUCKET_NAME="..."

./playbook.aws.yml -i inventory/aws_ec2.yml \
  --extra-vars "ansible_aws_ssm_region=${AWS_REGION}" \
  --extra-vars "ansible_aws_ssm_bucket_name=${X_AWS_SSM_BUCKET_NAME}"
```

### Azure — `az ssh` / AAD login

```sh
./playbook.azure.yml -i inventory/azure_rm.yml
```

The first play generates `./.az-ssh-config` via `az ssh config` (per VM, run on the controller); the main play uses it via `ansible_ssh_common_args`. The operator must have:

- `Virtual Machine Administrator Login` RBAC on the VM (sudo needed because `become: true`).
- Their public IP on the VM's NSG allow-list for `22/tcp`.
- An active `az login` session.

Full Azure VM prereq checklist: [docs/azure-prereqs.md](docs/azure-prereqs.md).

### Optional vars

| Variable         | Default                          | Purpose                                                           |
| ---------------- | -------------------------------- | ----------------------------------------------------------------- |
| `custom_hosts`   | `vm_tag_wireguard:app_wireguard` | Override the target host pattern.                                 |
| `upgrade_system` | `false`                          | Run `apt dist-upgrade + autoremove + autoclean` before deploying. |

```sh
./playbook.yml -i inventory/hcloud.yml --extra-vars upgrade_system=true
```

## Configuring `files/.env`

`files/.env` is the per-deployment configuration file read by the `linuxserver/wireguard` container. It is **not** generated — create or update it before running the playbook.

```sh
cat > files/.env <<'EOF'
SERVERURL=wg.example.com
ALLOWEDIPS=10.13.13.0/24,10.1.0.0/16,168.63.129.16/32
PEERS=alice,bob,charlie
EOF
```

| Variable     | Description                                                                                                                                                                                                                                                                                                    |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SERVERURL`  | Public DNS name (or IP) of the WireGuard server. Written into every peer config as the `Endpoint`.                                                                                                                                                                                                             |
| `ALLOWEDIPS` | Subnets routed through the tunnel in generated peer configs. Use the VNet/VPC CIDR plus `10.13.13.0/24` (tunnel subnet) and `168.63.129.16/32` (Azure DNS) if needed. Avoid `10.0.0.0/8` or broad ranges that overlap with clients' local LANs — macOS/Linux will silently prefer the connected-network route. |
| `PEERS`      | Comma-separated peer names. A keypair and config are generated for each on first run; existing configs are reused on subsequent runs unless parameters change.                                                                                                                                                 |

The container sets peer `DNS` to `10.13.13.1` (its own CoreDNS) by default. For Azure private DNS resolution, override the generated peer configs to use `DNS = 168.63.129.16` instead.

## GCP setup detail

`inventory/example.gcp.yml` is committed as a placeholder. Copy and edit:

```sh
cp inventory/example.gcp.yml inventory/gcp_compute.yml
$EDITOR inventory/gcp_compute.yml   # set projects: [your-real-project-id]
```

`inventory/gcp_compute.yml` is gitignored.

## Connectivity test

Before running a full converge, confirm Ansible can reach the host(s):

```sh
ansible vm_tag_wireguard:app_wireguard -i inventory/<cloud>.yml -m ping
```

(For Azure: run `./playbook.azure.yml --tags none` first to generate the SSH config, then `ansible ... -m ping -e "ansible_ssh_common_args='-F ./.az-ssh-config'"`.)

## Validation

```sh
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml
yamllint . && ansible-lint
```

## Authors

**Andre Silva** - [@andreswebs](https://github.com/andreswebs)

## License

This project is licensed under the [Unlicense](UNLICENSE).
