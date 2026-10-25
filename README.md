<!--
Copyright (C) 2026 Robert Wimmer
SPDX-License-Identifier: GPL-3.0-or-later
-->

# ansible-role-coredns-kubernetes

Deploy Kubernetes cluster DNS using the [official CoreDNS Helm chart](https://github.com/coredns/helm).
The role manages fresh installations and subsequent upgrades of its own release.
It does **not** adopt existing manifests, Helm releases without its management marker,
or distribution-managed DNS.

The default chart is **1.49.0**, containing **CoreDNS 1.14.7**. Chart and application
versions are distinct. Versions are pinned, never implicitly upgraded to latest.

## Requirements

- ansible-core >= 2.19.
- `kubernetes.core` >= 6.5.0 and `ansible.utils` >= 6.0.0; see [collections.yml](collections.yml).
- Helm **3**, plus a compatible [helm-diff](https://github.com/databus23/helm-diff) >= 3.9.12
  for deployment. The role inspects plugins but never installs them.
- Kubernetes Python client >= 24.2.0 and PyYAML on the execution host. The IP-address test
  does not require `netaddr`.
- An accessible Kubernetes API and appropriately authorized `kubeconfig`.
- Ready CNI/networking, workers with sufficient capacity, and an existing target namespace.

Helm 4 and dual-stack integration are not currently certified by this role.
The role does not install dependencies, change kubelets, configure CNI, or create/delete namespaces.

## Minimal usage

```yaml
- name: Manage cluster DNS
  hosts: localhost
  gather_facts: false
  vars:
    ansible_python_interpreter: "{{ ansible_playbook_python }}"
    coredns_service_ip: "10.32.0.254"
  roles:
    - githubixx.coredns
```

**Replace the example IP** with the address configured in every kubelet's
`clusterDNS`. Inspect kubelet configuration or your worker-role inventory.
For a previously configured cluster, also inspect its DNS Service address.
The role cannot reliably discover the intended kubelet setting or a universal
Service CIDR from the Kubernetes API. An allocated arbitrary Service IP is not
a safe default for cluster DNS.

Set `coredns_cluster_domain` to the kubelets' `clusterDomain` (default `cluster.local`).
The role refuses resource conflicts rather than taking over an existing DNS installation.

### Native lifecycle tags

| Selection | Behavior |
| --- | --- |
| No operation tag | Idempotent install/upgrade |
| `--tags coredns_present` | Explicit idempotent install/upgrade |
| `--tags coredns_template` | Render the chart without cluster access |
| `--tags coredns_rollback` | Restore the requested retained revision |
| `--tags coredns_absent` | Remove the owned release with acknowledgement |

```sh
ansible-playbook deploy.yml
ansible-playbook deploy.yml --tags coredns_template
ansible-playbook deploy.yml --tags coredns_rollback \
  -e coredns_rollback_revision=2
ansible-playbook deploy.yml --tags coredns_absent \
  -e coredns_allow_removal=true
```

Nondefault paths carry `never` and require their explicit operation tag.
Selecting multiple lifecycle tags or `never` is an error. Broad/inherited tags
must not enable rollback or removal. No partial-resource mutation tags are exposed.
Only complete lifecycle tags may be skipped; skipping mandatory/inherited tags is
unsupported and rejected. Molecule's `notest`, `molecule-notest`, and
`molecule-idempotence-notest` skips are allowed.
Skipping the selected nondefault operation does not fall through to deployment.

Prefer static `roles:` or `ansible.builtin.import_role`, which make operation tags
discoverable through `--list-tags`/`--list-tasks`. A dynamic `include_role` must itself
be reachable under the requested tag:

```yaml
- name: Include CoreDNS dynamically
  ansible.builtin.include_role:
    name: githubixx.coredns
  tags: always
```

Do not apply a blanket operation tag through `apply` to all role tasks.
Tags select work; they are not authorization controls.

`--check` does not deploy, uninstall, create preflight resources, or run rollback.
Deployment check mode requires API access and Helm diff. Rollback check mode
validates retained history without restoring it. Template check mode renders in
memory and predicts output-file changes without writing the manifests.

## Configuration

See [defaults/main.yml](defaults/main.yml) and [meta/argument_specs.yml](meta/argument_specs.yml)
for the complete typed interface.

| Variable | Default / purpose |
| --- | --- |
| `coredns_chart_version` | `1.49.0` |
| `coredns_chart_repository` | `https://coredns.github.io/helm` |
| `coredns_release_name` | `coredns` |
| `coredns_namespace` | `kube-system` |
| `coredns_service_ip` | Required for deployment/rendering |
| `coredns_cluster_domain` | `cluster.local` |
| `coredns_delegate_to` | `localhost`; inventory host running Helm/API modules |
| `coredns_kubeconfig` | Path on that host; honors `K8S_AUTH_KUBECONFIG` |
| `coredns_context` | Optional; honors `K8S_AUTH_CONTEXT` |
| `coredns_helm_binary` | `helm` |
| `coredns_timeout` | `300` seconds per bounded operation |
| `coredns_history_max` | `10`, minimum `3` |
| `coredns_values_template` | Empty; optional controller-side Jinja YAML mapping |
| `coredns_values` | `{}`; final chart overrides |
| `coredns_upstreams` | `1.1.1.1`, `9.9.9.9` |
| `coredns_allow_reduced_availability` | `false` |
| `coredns_allow_removal` | `false` |
| `coredns_rollback_revision` | Explicit positive retained revision for rollback |
| `coredns_template_path` | `~/coredns/template.yml` on the execution host |
| `coredns_preflight` | `false` |
| `coredns_probe_pod` | Empty; existing pod with `dig` |
| `coredns_probe_namespace` | `default` |
| `coredns_probe_container` | Empty; optional explicit container |
| `coredns_probe_image` | Kubernetes `agnhost:2.39` DNS-debugging image with `dig`, used for temporary preflight clients |
| `coredns_forwarding_probes` | `[]`; opt-in expected external/custom DNS answers |

Helm and Kubernetes operations run once on the delegate without privilege escalation.
Use the execution host's correct Python interpreter and `kubeconfig` file permissions.
Run the role once per release in a play; do not invoke concurrent operations on the
same release or run it through multiple serial batches/free strategy.

### Values precedence and Corefile customization

The public configuration interface is defined in [defaults/main.yml](defaults/main.yml).
[vars/main.yml](vars/main.yml) holds internal implementation data:
`coredns__operation_tags` defines the supported lifecycle tags, and
`coredns__baseline` supplies the initial Helm values. The double underscore marks
internal variables, not a separate user-facing configuration interface.

Ansible role vars take precedence over inventory and play variables, though
extra vars (`-e`) can still override them. Do not override these internal
variables directly. Customize chart settings through `coredns_values_template`
and `coredns_values` instead; the role explicitly merges these inputs over the
baseline, independently of Ansible's variable precedence.

The shared configuration path merges:

1. Role baseline.
2. Explicitly selected Jinja values-template mapping.
3. `coredns_values`.

Mappings merge recursively. **Lists replace**, including `servers`, plugins,
volumes, mounts, and tolerations. A requested missing/broken template is an error,
not a reason to silently use defaults. Conflicting Service IP inputs are rejected.
Overrides remain subject to the role's ownership and deployment safety checks.

The chart generates its own `ConfigMap/Corefile` from ordered `servers` and ordered
`plugins`. Each plugin supports `name`, optional `parameters`, and multiline
`configBlock`. The chart owns the ConfigMap, zone files, and deployment checksum,
so configuration changes roll pods and are included in Helm history.

Forwarding-probe definitions are retained in each revision's values and hashed
into a Deployment annotation, so a probe-only change is not lost to Helm diff.
Recovery checks the restored revision's probe contract. Helm resource-policy
`keep` and Helm hook annotations are rejected: acknowledged removal must not
orphan live DNS or turn ordinary DNS resources into unmanaged hook resources.

The role's built-in DNS configuration serves Kubernetes records and reverse
lookups, forwards other queries to the configured upstream resolvers, and caches
answers. It also provides health checks with a shutdown grace period, readiness
checks, Prometheus metrics, forwarding-loop detection, automatic configuration
reloads, and randomized answer ordering.

See the [examples guide](examples/README.md) for prerequisites, usage instructions,
and explanations of the example configurations:

- [examples/deploy.yml](examples/deploy.yml): minimal entry point.
- [examples/quad9-dot.yml.j2](examples/quad9-dot.yml.j2): equivalent authenticated Quad9 DoT.
- [examples/custom-zones.yml.j2](examples/custom-zones.yml.j2): multiple servers,
  multiline blocks, imported hosts snippet, and chart-owned zone file.

```yaml
coredns_values_template: /path/to/quad9-dot.yml.j2
coredns_values:
  resources:
    limits:
      memory: 256Mi
```

There is no separate raw-Corefile mode. Imports should use chart-owned zone files
where possible. Additional volumes/Secrets are supported by chart values, but
externally managed content is not restored by Helm rollback and changes to it
do not automatically change the chart's configuration checksum.

The upstream-compatible `pods insecure` directive generates pod-IP records without
verifying those IPs against pod objects. Consider `pods verified` or disabling pod
records for your needs; account for the corresponding memory/API implications.

### DNS-over-TLS

Quad9 uses `tls://9.9.9.9` with `tls_servername dns.quad9.net`. Permit TCP/853 egress.
The image needs a suitable trust bundle. For a private upstream, mount its CA
through chart volumes and specify the forward plugin's `tls` CA argument.
Do not disable certificate verification or silently fall back to plaintext.
The example does not enable DoT for downstream Kubernetes clients.

## Availability and rollout checks

Unlike the chart's one-replica/no-PDB defaults, the role defaults to two replicas,
`maxUnavailable: 0`, `maxSurge: 1`, PDB `minAvailable: 1`, preferred node separation,
and `system-cluster-critical` priority. It retains the chart's non-root/restricted
container security and health/readiness probes.

Fewer than two replicas, nonzero update unavailability, or missing/weakened PDB
protection require `coredns_allow_reduced_availability: true`. Otherwise the PDB
must select DNS pods through matching labels and protect at least one replica.
Percentage rounding follows Kubernetes semantics; with two replicas,
`maxUnavailable: 50%` protects one but `51%` does not. Set
`podDisruptionBudget: null` in chart overrides to disable it with acknowledgement.
Fixed-replica Deployment mode with
chart-owned configuration, dedicated ServiceAccount/RBAC, and enabled probes is
the supported first-release contract. HPA and proportional autoscaling are rejected.

Checks include Helm status, observed Deployment generation, all intended replicas,
drained old replicas, ready nonterminating pods, exact configuration/checksum,
Service/EndpointSlice identity and transport wiring, RBAC and PDB consistency.
Health checks surface relevant live drift; Helm diff does not guarantee repair
of every manual change.
Changing DNS Service IP/name or Deployment name/selector is a migration, not an upgrade.

A PDB protects voluntary evictions, not node failures or Deployment rolling
updates. Preferred affinity cannot guarantee node separation. Surge requires
spare capacity. These checks cannot guarantee zero DNS interruption.

### Functional DNS probes

Without a probe pod or preflight, only Kubernetes rollout readiness is verified;
the role explicitly reports this limitation. Supply an existing reachable pod
containing `dig` to query the DNS Service over both UDP and TCP:

```yaml
coredns_probe_namespace: diagnostics
coredns_probe_pod: dns-client
coredns_probe_container: dns
coredns_forwarding_probes:
  - name: answer.example.test.
    type: A
    expected: ["192.0.2.42"]
```

Internal checks use the API Service's actual address and the deployed cluster
domain. Forwarding checks are opt-in: choose stable names/expected answer tokens
appropriate to your environment. External outages must not unexpectedly become
a deployment gate. The execution identity needs pod-exec permission.

### Optional candidate preflight

Set `coredns_preflight: true` to deploy an isolated candidate release only when
the candidate actually differs. It uses independent names, Service address,
selectors, and RBAC. Existing-name collisions or unsafe isolation fail clearly.
If no probe pod is supplied, the role creates a temporary restricted DNS client.
Configure the probe image/egress policy appropriately for your environment.

Candidate health and enabled DNS probes run before production mutation.
Production is probed again after rollout. Candidate and temporary client resources
are cleaned up on success/failure, with cleanup errors reported.
Preflight needs extra capacity and cannot prove the eventual live rollout will succeed.
Advanced external/fixed-name configurations may be incompatible with isolation.

## Rollback and failures

Changed upgrades capture the healthy current revision. Helm atomic/wait handles
rollout failures; subsequent role-level health failures trigger restoration of
that captured revision. Restored configuration is checked against the old manifest,
not the rejected candidate. **The Ansible run still fails after successful recovery.**
Forwarding-probe definitions are retained as role metadata in Helm values, so
baseline/recovery checks use that revision's checks rather than newly requested
records that the old configuration may not serve.

Explicit rollback requires `--tags coredns_rollback` and a retained positive
`coredns_rollback_revision`. Lowering the configured history limit does not hide
older revisions that still exist; subsequent Helm mutations may prune them.
Failed/pending ordinary deployments require operator
inspection rather than an implicit takeover. A rollback creates a new revision;
the next ordinary run reconciles back to the currently declared configuration.

There is no prior revision on a first installation. Helm atomic failure may remove
that new release; if Helm succeeds and a later functional check fails, the role
retains it for diagnosis and reports failure. It never automatically uninstalls a
previously working release to simulate recovery.

The role uses native modules except narrowly scoped `command`/`argv` calls for
Helm version inspection, JSON history, retained-revision manifests, and rollback,
which lack equivalent modules. Rollback checks ownership of both current resources
and resources reintroduced by the target revision; Helm rollback alone does not
provide that safeguard.
No shell or kubectl wrappers are used. Protected diagnostic logs are not printed
because custom DNS plugins may log sensitive content.

## Testing

For chart rendering and input-validation checks without a working Kubernetes
cluster, see the [tests guide](tests/README.md). These checks do not require
Molecule or Vagrant, but Helm still needs access to the chart repository.

The Molecule scenario requires Vagrant with the `libvirt` provider, the Molecule
Vagrant driver, and the dependency roles available in the configured Ansible role
path. Run the following commands from this role's directory:

```sh
molecule test
molecule side-effect
molecule converge
molecule idempotence
molecule verify
```

`molecule test` prepares the guests, converges the role, checks idempotence, and
verifies the resulting cluster. `molecule side-effect` runs the additional
lifecycle exercises described below. Use the individual converge, idempotence,
and verify commands to repeat those checks against the existing guests.

The Vagrant/libvirt scenario uses nine isolated Ubuntu guests: assets,
three controllers, three etcd nodes, and two workers. Cilium is installed during
preparation; CoreDNS is deployed separately by this role. Guest tooling pins Helm 3.19.5
and Helm diff 3.14.0. Dependency role versions come from the configured role path;
record their actual revisions when reporting integration results.
Molecule prepares the assets guest with Kubernetes Python SDK 35.0.0 and the
required test dependencies. The scenario currently targets Linux x86_64 guests.

On the assets VM, the `vagrant` user can run `kubectl get nodes` without a
`--kubeconfig` argument. Preparation links `~/.kube/config`, kubectl's default
configuration path, to the protected `~/.kube/admin.kubeconfig` created by the
controller role. This keeps a single credential file rather than copying it or
depending on shell environment settings. Verification checks that plain
`kubectl` can list the expected nodes.

During preparation, Molecule disables and masks Ubuntu's automatic APT timers
and masks their services in these disposable guests only. Any APT job already
running is allowed up to ten minutes to finish before package installation starts.
This prevents background upgrades from racing the dependency roles for the
package-manager lock. No upgrade process is killed and no lock file is deleted;
the role does not change automatic-update settings on real cluster hosts.

`verify.yml` is read-only: fixtures are prepared beforehand. It checks dependency
services/API/node/CNI health, release revision, full convergence, security,
configuration/RBAC, Service/EndpointSlices, both transports across workers and
every replica, exact A/SRV/PTR/NXDOMAIN answers, search domains, health/readiness,
metrics, and temporary-resource cleanup.

The test upstream serves `example.test` with fixed answers and forwards other
names to the assets VM's upstream DNS servers, read from Ubuntu's
`/run/systemd/resolve/resolv.conf`. It does not use the VM's loopback DNS stub,
which is unreachable from an ordinary pod. A configuration checksum rolls the
upstream fixture when its DNS settings change.

Verification also resolves `www.google.com.` from both workers through the DNS
Service and every CoreDNS replica, over UDP and TCP. It requires `NOERROR` and
at least one valid IPv4 answer rather than comparing Google's changing IPs.
These public-name checks require working Internet DNS access from the test
cluster and fail if the external lookup fails; they are not silently skipped.
The role's own deployment probes remain explicitly configured separately.

`side_effect.yml` separately exercises changed configuration with preflight,
explicit rollback, invalid configuration with/without preflight, semantic-check
recovery, authenticated local DoT and authentication failures, guarded removal,
and reinstall. It also checks actually pruned history, foreign production
ownership, and a deliberately rejected recovery using temporary admission
controls, followed by explicit repair. A probe-only upgrade must persist its new
checks without restarting DNS pods. Worker-local samplers require successful
UDP and TCP answers throughout the successful upgrade, not just before/after it.
Its TLS upstream/CA and exact-answer probes are local fixtures; the separate
public-name checks in verification require Internet DNS. A bounded shell loop
inside disposable sampler pods is test-only; role tasks do not use shell wrappers.

The default sequence leaves guests available for lifecycle exercises. When finished,
use `molecule destroy` from this role to remove only its scenario guests.
