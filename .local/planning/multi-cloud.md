# Wireguard multi-cloud deployment plan

Adapt the existing `playbooks/wireguard/` (currently AWS-only) so the same role/playbook can target AWS, Azure, GCP, and Hetzner. Pattern is borrowed from `~/programs/git-repos.self/app-katas/_config/`.

## Source patterns observed in `app-katas/_config`

- **Single `ansible.cfg`** lists every cloud's inventory plugin in `[inventory] enable_plugins`. One default `inventory =` line points to whichever cloud is active; the others swap via editing or `-i`.
- **One inventory file per cloud** in `inventory/<cloud>.yml`. Each uses `keyed_groups` such that the same group name pattern (e.g. `vm_tag_wireguard`) resolves on every cloud — the playbook's `hosts:` line never changes.
- **One `requirements.yml`** with all cloud collections + role deps.
- **Playbook variants per cloud** only when a cloud needs cloud-specific vars or extra tasks. The bare `docker-app.yml` is the generic SSH path (Hetzner uses it as-is). `docker-app.aws.yml` adds `ansible_connection: aws_ssm`. `docker-app.gcp.yml` adds the GCP Ops Agent role.
- **Host pattern is uniform**: `vm_tag_{{ app }}` — every inventory must produce a group with that name. AWS via `tags["Name"]`, GCP via network tags, Hetzner via `labels`, Azure via `tags.Name`.

## Target layout for `playbooks/wireguard/`

```txt
playbooks/wireguard/
├── ansible.cfg                   # enable all 4 inventory plugins; no default inventory
├── requirements.yml              # amazon.aws, azure.azcollection, google.cloud, hetzner.hcloud, community.docker, andreswebs.docker_app
├── .ansible-lint.yaml            # existing
├── .yamllint                     # existing
├── .gitignore                    # existing
├── inventory/
│   ├── aws_ec2.yml               # existing — keyed_groups on tags["Name"]
│   ├── azure_rm.yml              # NEW — keyed_groups on tags.Name → vm_tag_*
│   ├── example.gcp.yml           # NEW — placeholder project ID, ADC auth (operator copies + edits)
│   └── hcloud.yml                # NEW — keyed_groups on labels (mirror app-katas)
├── playbook.yml                  # base: SSH path; usable as-is for Hetzner, GCP
├── playbook.aws.yml              # imports playbook tasks + sets ansible_connection=aws_ssm
├── playbook.azure.yml            # pre_tasks `az ssh config` shim + ansible_ssh_common_args
└── files/                        # unchanged (compose.yaml, .env)
```

Requirements

1. **Host pattern**: target group is `vm_tag_wireguard` (app-katas convention). AWS and Azure produce it natively via `tags.Name`. GCP produces it via the network-tag-keyed `vm_tag_*` group from app-katas. Hetzner's app-katas-style inventory (`keyed_groups: [{ key: labels }]`) produces `app_wireguard` instead — the playbook absorbs the difference with `hosts: vm_tag_wireguard:app_wireguard` (union pattern), keeping the Hetzner inventory identical to app-katas.
2. **No default inventory in `ansible.cfg`** — explicit `-i inventory/<cloud>.yml` beats remembering which one is active. (App-katas has a default; we deviate here because this playbook is more likely to be invoked against different clouds in the same week.)
3. **`requirements.yml` adds**: `azure.azcollection`, `google.cloud`, `hetzner.hcloud`. Flag in README that `azure.azcollection` requires extra pip packages from its own `requirements.txt`.
4. **Cloud-specific playbook variants**: AWS needs SSM connection (`playbook.aws.yml`); Azure needs `az ssh config` shim (`playbook.azure.yml`); GCP and Hetzner ride the base.
5. **Azure connection model**: direct SSH + AAD (`az ssh vm`) over public IP with NSG operator-IP allow-list, dynamic `azure_rm` inventory using operator's `az login`. Full detail in [agilitas-platform/.local/planning/wireguard-ops-playbook.md](/Users/andre/p41/current/agilitas/git-repos/agilitas-platform/.local/planning/wireguard-ops-playbook.md).
6. **`.envrc` stays cloud-specific** (currently AWS). Could be split per cloud later; out of scope for this round.

## Implementation phases

### Phase 1 — Base layout + Hetzner + GCP (no cloud-specific playbook)

- Add `azure.azcollection`, `google.cloud`, `hetzner.hcloud` to `requirements.yml`.
- Update `ansible.cfg` `enable_plugins` list; remove the default `inventory =` line.
- Add `inventory/hcloud.yml` — mirror app-katas: `plugin: hetzner.hcloud.hcloud` + `keyed_groups: [{ key: labels }]`. Operator labels the VM `app=wireguard` on the Hetzner side; the inventory exposes it as group `app_wireguard` (absorbed by the playbook's union `hosts:` pattern — see Decision #1).
- Add `inventory/example.gcp.yml` as a committed placeholder (mirror app-katas), with `auth_kind: application` (operator ADC), `projects: [your-gcp-project-id]`, and `keyed_groups: [{ key: tags["items"], prefix: vm_tag }]`. Operator copies to `inventory/gcp_compute.yml` (gitignored) and edits the project ID.
- Verify `playbook.yml` runs unchanged against Hetzner. For GCP, defer real-VM verification until a project ID is provided.

### Phase 2 — AWS variant cleanup

- The current `playbook.yml` is AWS-flavored (assumes SSM via `--extra-vars`). Either:
  - (a) Keep the base playbook cloud-agnostic and move the SSM `--extra-vars` invocation into a dedicated `playbook.aws.yml` (matches app-katas), or
  - (b) Leave SSM connection vars as `-e` flags from the README (current style; less DRY).
- Recommended: (a). Matches the established pattern; one fewer thing to remember at invocation time.

### Phase 3 — Azure

- Add `inventory/azure_rm.yml` with `keyed_groups: [{ prefix: vm_tag, key: tags.Name }, { prefix: role, key: tags.role }]`. Filter by `include_vm_resource_groups:`. The VM should be tagged with both `Name=wireguard` and `role=wireguard` so the inventory emits `vm_tag_wireguard` (for this playbook) _and_ `role_wireguard` (for the ops-repo plan).
- Add `playbook.azure.yml`: imports the base playbook tasks and adds a `pre_tasks:` step that runs `az ssh config --resource-group $RG --name $VM --file ./.az-ssh-config`, plus `ansible_ssh_common_args: "-F ./.az-ssh-config"` on the play.

### Phase 4 — README rewrite

- Document `-i inventory/<cloud>.yml` per cloud, and which `playbook.<cloud>.yml` to invoke (or the base `playbook.yml` for GCP/Hetzner).
- Document the `azure.azcollection` extra pip install gotcha.
- Document the host-pattern contract: target is `vm_tag_wireguard`; Hetzner uses union `vm_tag_wireguard:app_wireguard`.

## Non-goals (deferred)

- Splitting `.envrc` per cloud.
- Spinning up the VMs themselves (Terraform / cloud-specific provisioning). This plan covers only configuration of existing VMs.
- Per-cloud Molecule scenarios. The base playbook should pass a `--check --diff` ladder; full Molecule is out of scope.
