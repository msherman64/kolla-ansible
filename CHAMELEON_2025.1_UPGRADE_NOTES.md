# Chameleon kolla-ansible 2025.1 upgrade — deferred items

This file records work that was intentionally **not** done during the
`stable/2025.1` rebase of the Chameleon fork, so it is not lost when the
upgrade branches are merged. It is fork-local (not upstream kolla-ansible).

## Deferred: dev-mode source binds — coupled to the companion kolla *images* repo

**Decision:** defer to the kolla-images upgrade. **No change made in this repo.**

### What
Several Chameleon roles still bind a dev checkout over the installed package's
`site-packages` directory, the pre-2025.1 dev-mode style:

| Role | Variable / line | Bind |
| --- | --- | --- |
| doni | `doni_dev_mode` volumes | `…/doni/doni → …/venv/lib/python<v>/site-packages/doni` |
| tunelo | `tunelo_dev_mode` volumes | `…/tunelo/tunelo → …/site-packages/tunelo` |
| zun | `zun_dev_mode` volumes | `…/zun/zun → …/site-packages/zun` |
| neutron | `neutron_server_plugin_volumes_raw` | per-plugin `…/neutron-plugins/<name>/<pkg> → …/site-packages/<pkg>` |
| nova | `nova_plugin_volumes_raw` | per-plugin `…/nova-plugins/<name>/<pkg> → …/site-packages/<pkg>` |
| horizon | `horizon_blazar_dev_mode` volumes | `…/blazar-dashboard/blazar_dashboard → …/site-packages/blazar_dashboard` |

Upstream 2025.1 moved **all core** dev-mode binds from `…/site-packages/<proj>`
to `<repo> → /dev-mode/<proj>`. The `/dev-mode/` directory is `pip install -e`'d
by the **kolla images** (companion repo), not by kolla-ansible.

### Why deferred
1. **Production-safe as-is.** Every bind is gated `… if <svc>_dev_mode | bool
   else ''`, and `kolla_dev_mode` defaults to `"no"`. With dev-mode off the
   entries render empty and are dropped — production deploys/upgrades are
   unaffected. This only matters to developers running `*_dev_mode: yes`.
2. **The mechanism lives in the images repo (out of scope).** Whether
   `/dev-mode/<proj>` works depends on the image performing the editable
   install. Migrating kolla-ansible alone, without the matching image change,
   would break dev-mode rather than fix it. `distro_python_version` still
   exists in 2025.1 so the old templates still render (no playbook error), but
   the overlay path may no longer match the image's venv layout.
3. **No upstream precedent for plugin dev binds.** `/dev-mode/` is only ever
   used for *top-level* projects upstream. The neutron/nova plugins and
   blazar-dashboard are sub-packages with no upstream pattern; converting them
   is purely speculative and requires the image to editable-install each plugin.

### Migration recipe (do this WITH the kolla-images 2025.1 upgrade)
1. **Top-level projects** (doni, tunelo, zun): change the bind to
   `kolla_dev_repos_directory ~ '/<proj>:/dev-mode/<proj>'` and ensure the
   corresponding image `pip install -e /dev-mode/<proj>` (mirror what upstream
   images do for nova/neutron/etc.).
2. **Plugins / sub-packages** (neutron-plugins, nova-plugins, blazar-dashboard):
   no upstream pattern — decide the convention image-side first (e.g. bind each
   plugin repo to `/dev-mode/<plugin>` and have the image editable-install every
   `/dev-mode/*`), then update `*_plugin_volumes_raw` / the blazar-dashboard
   bind to match.
3. Verify by setting the relevant `*_dev_mode: yes` and confirming the source
   tree is actually imported inside the container.

## Other Hop-2 verification items (need a real cloud, not code changes)
- **snmp-exporter**: generator migrated from `docker run` to
  `kolla_container_engine run` — verify the generator runs under podman.
- **zun**: `check-containers` now uses the shared `service-check-containers`
  role; confirm `zun_docker_common_options` overrides still apply via
  handlers/pull/bootstrap. (zun is removed at Hop 3.)
- **Staging deploy / SLURP validation**: full deploy + skip-level upgrade smoke
  test on staging.
