# Plan: Upgrade Chameleon's kolla-ansible fork — 2023.1 → 2024.1 → 2025.1 (2026.1 deferred)

## Context

Chameleon runs a fork of `kolla-ansible` (`ChameleonCloud/kolla-ansible`). The production
branch `chameleoncloud/2023.1` carries the Chameleon customization set on top of upstream
2023.1 (Antelope), which is now End-of-Life upstream. The goal is a repeatable procedure to
**rebase a curated Chameleon patch set forward onto each newer upstream release**, dropping
patches that no longer apply and absorbing upstream breaking changes hop-by-hop.

The chosen sequence follows OpenStack **SLURP** (Skip-Level Upgrade Release Process): each
release is a `.1` SLURP release, so every hop is a sanctioned skip-level jump but spans **two**
OpenStack releases of upstream change:

| Hop | kolla-ansible tags | Series spanned | Status |
|-----|--------------------|----------------|--------|
| 0: catch up to EOL | → `2023.1-eol` | 2023.1 (Antelope) | **committed** |
| 1: 2023.1 → 2024.1 | `2023.1-eol` → latest `18.x` | Bobcat (2023.2) + Caracal (2024.1) | **committed** |
| 2: 2024.1 → 2025.1 | `18.x` → `stable/2025.1` (`20.x`) | Dalmatian (2024.2) + Epoxy (2025.1) | **committed** |
| 3: 2025.1 → 2026.1 | `20.x` → `22.x` | Flamingo (2025.2) + 2026.1 | **deferred** (Zun removed upstream — see Open Decisions) |

**Decisions baked into this plan:**
- Target the full path through **2025.1**; **2026.1 is a later phase** (upstream removes Zun in 2026.1).
- Workflow: rebase the curated patch set onto upstream releases, creating
  **`chameleoncloud/2024.1`** and **`chameleoncloud/2025.1`** (matching `chameleoncloud/2023.1`).
- **Ignore the existing `backport/*` staging branches entirely** — start fresh from a curated set.
- Scope: **fork code only.** Companion `kolla` custom image builds (doni, tunelo, blazar) are a
  separate next task.

## Approach: rebase a curated patch set

Use **rebase / cherry-pick replay**, not merge. The Chameleon delta against `2023.1-eol`
(sha `2714e56`) is a well-defined linear set. Replaying it onto each upstream release makes
per-patch conflict resolution explicit and lets obsolete patches be dropped cleanly. Derive
the ordered patch list from `2023.1-eol..chameleoncloud/2023.1`.

## Environment / access facts (for the executor)

- **Remotes** (on a machine with full network):
  - `upstream` = `https://opendev.org/openstack/kolla-ansible.git` (or GitHub mirror `openstack/kolla-ansible`)
  - `chameleon` = `https://github.com/ChameleonCloud/kolla-ansible.git`
  - `git fetch upstream --tags && git fetch chameleon`
- This sandbox blocks `opendev.org`, `github.com` git, and `docs.openstack.org`; only
  `api.github.com` is reachable, so all research below was done via the REST API. Re-verify
  exact point-release tags on the real remote.
- **Upstream branch retention:** only `master`, `stable/2025.1`, `stable/2025.2`,
  `stable/2026.1`, `unmaintained/2024.1` survive as branches. **2023.1 and 2023.2 exist only
  as tags** (`2023.1-eol`, `16.x`, `17.x`); 2024.1 is `unmaintained/2024.1` + `18.x` tags. →
  Rebase Hop 1 onto the **latest `18.x` tag**; rebase Hop 2 onto **`stable/2025.1`**.
- Version map: `16=2023.1`, `17=2023.2`, `18=2024.1`, `19=2024.2`, `20=2025.1`, `21=2025.2`, `22=2026.1`.

## Hop 0 — bring `chameleoncloud/2023.1` current with `2023.1-eol`

`chameleoncloud/2023.1` is **7 commits behind** the final upstream `2023.1-eol` tag (late EOL
fixes never pulled in). Before any forward rebase:
1. Fetch `upstream` tags; identify the 7 commits in `chameleoncloud/2023.1..2023.1-eol`.
2. Merge/cherry-pick the still-relevant ones onto `chameleoncloud/2023.1` so the baseline is
   `2023.1-eol` + Chameleon patches, fully current.
3. This also establishes the clean `2023.1-eol` base for deriving the patch set in Hop 1.

## Patch-set curation (done once, before Hop 1)

Walk `2023.1-eol..chameleoncloud/2023.1` and classify every patch into **KEEP / DROP / SQUASH /
REDO**. Concrete dispositions:

**DROP (per direction / obsolete):**
- **LetsEncrypt** — drop the custom "Add support for LetsEncrypt-managed certs" patch; adopt
  upstream's built-in `letsencrypt` role instead (confirm feature parity).
- **Kuryr** — drop kuryr-related bits (e.g. "[zun] skip kuryr precheck when using k8s").
- **Cyborg** — drop all cyborg patches: "[cyborg] Expose to haproxy", "[cyborg] Enable dev
  mode", "[zun] Add cyborg client config", "Add cyborg_agent_environment customization".
- **EOL pin workaround** — "deps: pin to eol tag, branch is gone" (only needed for the dead
  2023.1 branch; the new releases have live branches/tags).
- **Merge commits** — e.g. "Merge remote-tracking branch 'upstream/unmaintained/2023.1'",
  "Merge pull request #…" — these drop naturally in a cherry-pick replay.

**REDO (version-specific — recreate against the new release, don't carry):**
- Image-name plumbing tied to 2023.1: "update chi added images for new names in 2023.1",
  "[prometheus] update image names for infra renaming", "specify correct snmp generator image
  key", "fix path for snmp-exporter-generator", "[neutron] fix neutron-wireguard-agent image
  name". Re-derive image refs for each target release.
- "Update horizon local settings for Django 4" — re-base on the newer Horizon's settings.
- Ironic-Inspector workarounds: "[Ironic-Inspector] slim down non-standalone case", "ironic:
  skip check when ironic not in standalone mode" — Inspector is deprecated in 2025.1; re-evaluate
  (likely droppable by Hop 2).
- container_engine plumbing: "[doni] specify container_engine for kolla_toolbox", "set
  container_engine for kolla_container_facts" — upstream gained podman/container_engine support
  in this window; verify whether still needed.

**SQUASH (fold fixups into their parent feature during rebase, don't carry as separate patches):**
- "[zun] fixup infinite loop", "[prometheus] fixup merge conflict", "[neutron] fixup adding
  networking-wireguard", "Fixup oidc feature compat", "Fix missing when condition", duplicate
  "set BASEPATH for editable install".

**KEEP (Chameleon-owned features to carry forward):**
- Identity/OIDC: WebSSO default redirect, OIDC assertions as JSON, allowed OIDC redirect URLs,
  templated OIDC metadata, single-IdP default discovery, Keystone vhost config, wsgi cleanup.
- Horizon: prefix redirect, openrc template override, custom clouds YAML.
- Prometheus monitoring stack: pushgateway, SNMP exporter (+defaults), JupyterHub exporter,
  IPMI exporter, ironic exporter (disabled by default), custom alertmanager templates,
  openstack-exporter flags, hostname-labeled cadvisor/node_exporter metrics.
- Networking: WireGuard agent + plugin mounts, NGS toggle, OVS facts handling.
- Reservation/bare-metal: `doni` role (+ k8s support), `blazar` host-agg/dashboard (+ k8s support).
- Edge/Kubernetes: `k3s` role, Calico CNI (+ GlobalNetworkPolicies, worker NodeIP, CRD waits),
  NVIDIA device plugin, smarter-device-manager, edge serial/device mappings.
- `tunelo` channel service (+ pwd generation).
- Zun customizations **minus cyborg/kuryr**: local mount, docker option overrides, zun-compute
  toggle, zun-compute-k8s config, kubeconfig override/skip.
- Platform/glue: `etcd` ARM support, `manila-nfs-ganesha` role, kolla_toolbox nested
  module_args + `--check/--diff`, post-deploy play hook, admin openrc dir customization,
  heat domain override, heat_auth_encryption_key handling, fluentd parsing for
  neutron-ironic-agent and zun logs, CHI service bootstrap oneshot policy.

Produce a patch-disposition table (every patch → KEEP/DROP/SQUASH/REDO + reason; no silent drops).
This curated KEEP set is what gets rebased in Hops 1 and 2.

## Custom roles (carry intact — self-contained, low conflict risk)

`ansible/roles/{k3s, doni, tunelo}` (and `manila-ganesha` glue) are new directories with few
upstream collisions; replay as-is. (The custom `letsencrypt` role is dropped in favor of upstream.)

## Per-hop upstream breaking changes to absorb

### Hop 1 — 2023.1 → 2024.1 (Bobcat + Caracal; `2023.1-eol` → latest `18.x`)

Mandatory / breaking:
- **Inventory: `[haproxy]` group renamed to `[loadbalancer]`** — update inventory before run.
- **`loadbalancer` role refactor:** `haproxy_processes`/`nbproc` removed, `nbthread` only.
- **`enable_haproxy_memcached` now defaults to `no`** — direct memcached access.
- **RabbitMQ HA + durable queues default on**, `cluster_partition_handling=pause_minority`
  (needs 3-node cluster); new `kolla-ansible rabbitmq-reset-state` helper.
- **`rabbitmq_hipe_compile` removed.**
- Removed/deprecated: **Sahara, MongoDB removed; Ceph deploy deprecated; Monasca Grafana fork deprecated.**
- **Ansible 8+ (ansible-core 2.15/2.16) required.** Host OS: Rocky Linux 9 recommended,
  Debian Bookworm added, CentOS Linux 8 / CentOS 7 gone.

Conflict hotspots vs. KEEP set: **loadbalancer/HAProxy** (Chameleon haproxy-config-hiding,
cyborg-expose now dropped), **RabbitMQ** defaults, **Keystone/Horizon OIDC** (memcached access).

### Hop 2 — 2024.1 → 2025.1 (Dalmatian + Epoxy; `18.x` → `stable/2025.1`)

Mandatory / breaking:
- **Python 3.8/3.9 dropped (min 3.10).**
- **Swift deprecated** (removal in 2025.2).
- **Ironic ecosystem overhaul:** Bifrost deprecated; Inspector defaults disabled;
  `ironic_neutron_agent` defaults to `no` (manual cleanup) — re-evaluate Chameleon Ironic/Doni
  workarounds (drop the Inspector REDO patches here if obsolete).
- **prometheus-msteams removed** — check Chameleon Prometheus stack (not used by Chameleon exporters).
- **OpenEuler host OS dropped.**
- **RabbitMQ:** quorum queues for transient/fanout default on; feature flags auto-enabled
  (`rabbitmq_feature_flags` no longer needed); **NEW `kolla-ansible upgrade-rabbitmq-target`
  command — required step for SLURP.**
- **MariaDB/ProxySQL:** SSL MariaDB↔ProxySQL; ProxySQL defaults reverted to upstream
  (timeout/retry change for multinode); socat-based cluster check.
- OVN SB DB relay auto-deployed on upgrade (toggle `enable_ovn_sb_db_relay`).
- HAProxy TLS hardened to Mozilla "modern" (revertable to "legacy").

Conflict hotspots vs. KEEP set: **Ironic/Doni**, **Prometheus** exporters, **MariaDB/ProxySQL**,
**OVN/Neutron** (WireGuard agent), **Zun** (still present in 2025.1).

### Hop 3 — 2025.1 → 2026.1 (DEFERRED)

Documented for awareness; **not executed** here:
- **Zun + Kuryr removed upstream** ("Zun broken in 2026.1") — conflicts with Chameleon's Zun
  KEEP set. This is the deferral reason. (Kuryr/cyborg already dropped in curation.)
- Ironic **Inspector fully removed** (var renames `ironic_inspector_*` → `ironic_*`).
- VMware drivers removed; **InfluxDB v1 + Telegraf removed**; Venus removed; legacy iptables dropped.
- **ProxySQL becomes mandatory** with MariaDB; HAProxy+clustercheck backend dropped.
- **`innodb_log_file_size` default 96MB → 2GB** (plan disk space).

### SLURP procedure (validation context for the rebased branches)

Upstream `doc/source/user/operating-kolla.rst` (N→N+2 supported). Per hop: backup `/etc/kolla`;
install matching `kolla-ansible` + **bump ansible-core to the target's supported version**;
`kolla-ansible install-deps`; reconcile `globals.yml`/`passwords.yml`/inventory;
`kolla-ansible pull && prechecks`; **`kolla-ansible upgrade-rabbitmq-target`** (RabbitMQ can't
skip majors, Hop 2+); `kolla-ansible upgrade`. Avoid `--limit` (known Nova bugs); ensure
`log_bin_trust_function_creators=1` for pre-provisioned DBs.

## Execution per hop (fork code)

1. **Pick upstream base:** Hop 1 → latest `18.x` tag; Hop 2 → `stable/2025.1`.
2. **Create branch** `chameleoncloud/<release>` from that base.
3. **Replay the curated KEEP set** in order (`git cherry-pick` / `git rebase --onto`), with
   fixups squashed and REDO patches recreated against the new tree.
4. **Resolve conflicts by hotspot** (Hop 1: loadbalancer rename, RabbitMQ, memcached/OIDC;
   Hop 2: Ironic/Doni, Prometheus, MariaDB/ProxySQL, OVN/WireGuard).
5. **Reconcile config & inventory** to new upstream defaults (haproxy→loadbalancer, RabbitMQ HA,
   memcached, Ansible version pins, image refs).
6. **Validate** (below), set `chameleoncloud/<release>` as the new baseline, repeat for the next hop.

Per-hop deliverables: the rebased `chameleoncloud/<release>` branch, the patch-disposition table,
and an updated reference `globals.yml`/inventory diff for operators.

## Critical files / areas (representative)

- Inventory templates + `ansible/group_vars/all.yml` (haproxy→loadbalancer, RabbitMQ, memcached,
  `openstack_release`/image tags, Ansible version).
- `ansible/roles/loadbalancer/*` (Hop 1), `ansible/roles/rabbitmq/*`, `ansible/roles/mariadb/*` + ProxySQL (Hop 2).
- `ansible/roles/ironic/*` + Chameleon `doni`, `blazar` roles (Hop 2 Inspector/neutron-agent defaults).
- Chameleon `prometheus` exporter additions (Hop 2 msteams removal).
- OIDC patches in `keystone`/`horizon` roles (Hop 1 memcached access; Hop 2 Horizon/Django).
- Self-contained roles `ansible/roles/{k3s,doni,tunelo}` (carry intact).
- `requirements.txt` / `doc/` / CI for the ansible-core bump per hop.

## Verification

- **Per-branch static:** `git log --oneline <basetag>..chameleoncloud/<release>` matches the
  curated KEEP set; `tox -e pep8` / yamllint / ansible-lint clean; `kolla-ansible prechecks` passes.
- **Patch accounting:** the disposition table is complete (KEEP/DROP/SQUASH/REDO, with reasons).
- **Deploy test:** on a staging cloud, run the SLURP procedure (incl. `upgrade-rabbitmq-target`
  at Hop 2) and confirm control plane healthy (`openstack endpoint/service list`, RabbitMQ
  cluster, MariaDB/Galera, OVN northd/relay). Smoke test: boot instance, attach volume, floating
  IP, and Chameleon paths — Blazar reservation, Doni enrollment, OIDC WebSSO login, k3s edge deploy.
- **Between hops:** the cloud must be healthy on 2024.1 before starting Hop 2.

## Out of scope / next tasks

- Companion `kolla` custom image builds (doni, tunelo, blazar) — separate task; this plan assumes
  matching images become available.
- Full operational deployment runbook beyond the validation above.
- **Hop 3 (2026.1)** execution.

## Open decisions (before Hop 3)

- **Zun at 2026.1:** upstream removal forces a choice — maintain Zun out-of-tree as
  Chameleon-owned roles, migrate workloads off Zun, or drop it. Revisit after 2025.1 is stable.
- Re-confirm Ironic Inspector removal impact on Doni and InfluxDB/Telegraf removal vs. Chameleon
  monitoring at that time.
