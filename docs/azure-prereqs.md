# Azure VM prerequisites — Wireguard playbook

<!-- markdownlint-disable MD029 -->

Handover doc for the ops/infra team. Confirm each item before the [playbook.yml](../../playbook.yml) is run against the Azure VM. Most items live on the Azure infrastructure side (Terraform), not in this Ansible repo.

Cross-reference: [agilitas-platform/.local/planning/wireguard-ops-playbook.md](/Users/andre/p41/current/agilitas/git-repos/agilitas-platform/.local/planning/wireguard-ops-playbook.md).

## Must-have before the playbook can connect

1. **AAD SSH extension installed on the VM** (`AADSSHLoginForLinux`). Without it, `az ssh vm` fails. Install once via Terraform or CLI:

   ```sh
   az vm extension set \
     --resource-group "${RG}" --vm-name "${VM}" \
     --name AADSSHLoginForLinux \
     --publisher Microsoft.Azure.ActiveDirectory
   ```

2. **RBAC role for the operator**: `Virtual Machine Administrator Login` (not just `User Login`) on the VM scope. The playbook does `become: true become_user: root`, which needs sudo. `User Login` does not grant sudo.

3. **NSG inbound rule**: `22/tcp` from the operator's public IP (or operator allow-list prefix).

4. **VM tags**: `Name=wireguard` and `role=wireguard`. The `azure_rm` inventory keys on `tags.Name` (for `vm_tag_wireguard` group, used by this playbook) and `tags.role` (for `role_wireguard` group, used by the ops-repo plan). Without both tags the host will not land in the expected group.

## Must-have before the playbook can converge

5. **Outbound internet from the VM** — needs to reach `apt` mirrors (system upgrade), Docker Hub, and `lscr.io` (linuxserver image pull). If the spoke has an Azure Firewall filtering egress, allow those FQDNs.

6. **Ubuntu LTS image** — the playbook is apt-based and assumes systemd. Stock Ubuntu 22.04 or 24.04 cloud image is appropriate; provides `python3`, the WireGuard kernel module, and `apt` out of the box.

7. **Static public IP** (recommended). WireGuard clients connect by hostname; if the IP drifts across deallocate/reallocate cycles, the DNS record breaks.

## Must-have for the WireGuard service to function (post-converge)

8. **NSG inbound rule**: `51820/udp` from `0.0.0.0/0`, or whatever client allow-list applies.

9. **DNS A record** pointing to the VM's public IP. For the Agilitas dev VM, set `SERVERURL=wg.agilitas.sandbox.particle41.ninja` → `48.216.169.8` in [files/.env](../../files/.env). Confirm that record exists and resolves to the VM before generating client configs.

## Worth verifying

- **IP forwarding at the NIC level**: Azure normally requires "IP forwarding" on the NIC when the VM forwards packets with foreign source IPs. This playbook MASQUERADEs inside the container, so packets leaving the NIC carry the VM's own IP — NIC-level IP forwarding is **not** required and can stay disabled (which is helpful in subscriptions with an ALZ deny policy on NIC IP forwarding).
- **MTU**: WireGuard over Azure's underlay sometimes needs MTU 1380 (vs the WG default of 1420). Not a prerequisite, but if peers connect but throughput is poor, this is the first knob.
- **Custom hosts var**: the playbook expects `--extra-vars custom_hosts=...`. With the `vm_tag_wireguard` group, invoke with `--extra-vars custom_hosts=vm_tag_wireguard`. Alternatively, change the default in [playbook.yml](../../playbook.yml) from `'servers'` to `'vm_tag_wireguard'`.

## Handled by the playbook (no manual prep)

- Docker install — handled by the `andreswebs.docker_app` role's dependency chain.
- Sysctls (`net.ipv4.ip_forward`, `src_valid_mark`, `proxy_arp`) — set by the container via the `sysctls:` block in [files/compose.yaml](../../files/compose.yaml).
- WireGuard kernel module — loaded on demand from `/lib/modules` (mounted into the container).
- systemd unit creation and container startup — handled by the role.

## Pre-flight checklist (suggested)

Before running the playbook, the operator should be able to answer "yes" to all of:

- [ ] `az ssh vm --resource-group "${RG}" --name "${VM}"` connects without prompting for a password.
- [ ] `sudo -n true` succeeds inside that SSH session.
- [ ] `az resource list --resource-group "${RG}" --query "[?tags.Name=='wireguard' && tags.role=='wireguard']"` returns the VM.
- [ ] `dig +short ${SERVERURL}` returns the VM's public IP.
- [ ] NSG rules for `22/tcp` (operator IP) and `51820/udp` (clients) are present.
- [ ] `az vm extension list -g "${RG}" --vm-name "${VM}" --query "[?name=='AADSSHLoginForLinux']"` is non-empty.

If all six pass, the playbook should converge on first run.
