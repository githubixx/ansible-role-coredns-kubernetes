<!--
Copyright (C) 2026 Robert Wimmer
SPDX-License-Identifier: GPL-3.0-or-later
-->

# CoreDNS role progress

## Approved scope

Use the official CoreDNS Helm chart for fresh installations and subsequent
role-managed upgrades. Do not adopt existing manifests or distribution-managed
DNS. Preserve the existing Cilium role.

Chart baseline: **1.49.0**, CoreDNS **1.14.7**.

Lifecycle selection uses native tags, not an action/state variable:

| Selection | Operation |
| --- | --- |
| Ordinary execution / `coredns_present` | Idempotent installation/upgrade |
| `coredns_template` | Render without cluster access |
| `coredns_rollback` | Restore an explicit retained revision |
| `coredns_absent` | Acknowledged removal of the owned release |

Nondefault operations require explicit selection. Broad/inherited tags must not
trigger them. Conflicting operation tags and unsafe partial skips fail before
mutation. Mandatory checks remain part of each entry point.

## Configuration and safety contract

- Chart-native ordered servers/plugins, multiline blocks, imports, zone files,
  and optional controller-side Jinja values templates; no separate raw-Corefile
  mode or copied chart manifests.
- Merge baseline, selected template mapping, then explicit values mapping.
  Lists replace. Explicitly requested missing/broken templates fail.
- Require the DNS Service IP; it must match kubelet `clusterDNS`. Domain defaults
  to `cluster.local` and must match kubelet `clusterDomain`.
- Default to `kube-system`, release `coredns`, Service/label `kube-dns`, two
  replicas, zero-unavailable updates, PDB minimum one, preferred node separation,
  and critical priority. Reduced availability requires acknowledgement.
- Preserve plain Cloudflare/Quad9 forwarding; document opt-in authenticated
  Quad9 DoT. External forwarding probes are configurable and opt-in.
- Check ownership, immutable identity, effective settings, full rollout,
  EndpointSlices, and dedicated RBAC. Never force replacement or take ownership.
- Candidate preflight is opt-in and isolated. Optional existing-pod DNS probes
  test real UDP/TCP answers. Candidate/probe resources are created outside
  Molecule verification and cleaned up visibly.
- Use native modules; narrow Helm CLI exceptions cover capabilities without
  native modules. No shell or kubectl wrappers.
- Atomic upgrade recovery plus rollback to the captured healthy revision after
  role-level check failure. Recovery must not hide the attempted rollout failure.
- A first install has no revision to restore. Atomic install failure can remove
  it; later functional-check failure retains the new release for diagnosis.
- Guard removal separately and never delete namespaces.
- Initial baseline: ansible-core >=2.19, kubernetes.core >=6.5.0, Helm 3, and
  compatible Helm diff. Do not claim untested Helm 4 or dual-stack support.

PDBs do not guarantee availability through node failure or govern Deployment
updates. Preferred placement is not mandatory separation. Zero-unavailable
updates require spare capacity. Preflight/readiness/recovery do not guarantee
uninterrupted DNS.

## Milestones

| Milestone | Status |
| --- | --- |
| 1. Progress record and typed/tagged role contract | Implemented |
| 2. Shared values, pinned-chart rendering, Corefile examples | Implemented; offline tests passing |
| 3. Preconditions, ownership and identity safeguards | Passing, including negative and rollback-target-only ownership cases |
| 4. Native lifecycle and shared health checks | Converge and idempotence passing |
| 5. Optional DNS probes and isolated preflight | Passing with controlled plain/DoT upstreams and continuous rollout sampling |
| 6. Explicit rollback and automatic recovery | Passing, including induced recovery failure and explicit repair |
| 7. Reused multi-node Molecule setup and controlled fixtures | Preparation passing on all nine guests |
| 8. Read-only comprehensive verification | Passing with zero changes |
| 9. Configuration/lifecycle/recovery/negative exercises | Full lifecycle suite passing |
| 10. Documentation and executed acceptance validation | Complete; final checks passed and disposable guests destroyed |

The dependency order is contract -> values -> safeguards -> lifecycle.
Infrastructure follows the contract independently. Probes/recovery follow
lifecycle; verification needs lifecycle and infrastructure; lifecycle exercises
need probes/recovery and verification. Final acceptance follows all milestones.

## Required verification

- Lint, syntax, offline rendering and negative tag/input checks.
- Multi-node Vagrant/libvirt default sequence:
  prepare -> converge -> idempotence -> verify.
- Read-only verify: control-plane/CNI readiness, admin access and artifacts,
  exact chart/version/revision, full replicas, identity/security/PDB/RBAC,
  Service and ready EndpointSlices, Corefile/checksum, HTTP probes/metrics,
  cross-worker UDP/TCP A/SRV/PTR/NXDOMAIN and per-replica DNS.
- Controlled plain and authenticated DoT upstreams; no mandatory Internet DNS.
- Separate mutation-heavy exercises: plain-DNS and DNS-over-TLS Corefiles, custom
  zones/domain/imports, precedence, upgrade, rollback, preflight failure,
  automatic and semantic-check recovery, TLS failures, safeguards, cleanup,
  guarded removal/reinstall, check mode, tag routing/inheritance/skipping.
- Capture bounded diagnostics without leaking credentials.
- Record actual executed versions/results and blockers; never infer successful
  integration from successful rendering alone.

## Evidence and implementation notes

- Inspected the role skeleton, Cilium scenario, CoreDNS configuration examples,
  published chart/templates, and installed kubernetes.core module capabilities.
- Existing user changes were preserved; no commits were created.
- The shared direnv toolchain preflight passed.
- Local Helm is 3.19.5; no Helm plugins are currently installed. Do not alter
  shared Helm state just to satisfy tests.
- Repository implementation began after plan approval. Executed integration
  results and fixture corrections are recorded below.
- Real upstream-chart offline tests passed, including default output, documented
  values templates, invalid inputs, availability acknowledgement, native tag
  conflicts, missing removal acknowledgement, and dynamic role inclusion.
- `ansible-lint --offline .`, YAML lint, and prepare/verify/lifecycle syntax checks
  passed. These checks do not establish rollout/recovery behavior.
- All nine scenario guests were created and are reachable through Vagrant.
  Initial preparation failed because the running playbook did not yet include
  the assets-host kubectl dependency. Preparation was corrected and restarted.
- Package downloads in the disposable guests now have bounded timeouts/retries.
- Necessary CLI exceptions cover Helm version inspection, history, retained-
  revision manifests, and rollback. All calls use `command` with `argv`.
- Forwarding-probe definitions are retained in Helm values; restored revisions
  use their own functional-check contract, not the rejected candidate's checks.
- Real-chart offline acceptance passed again after the latest safeguard changes.
  Role/scenario Ansible lint and targeted YAML lint passed. Full-directory YAML
  lint additionally reports an existing trailing blank line in the user's
  release workflow; that unrelated file was not modified.
- Fixture preparation exposed stale in-memory etcd CA state after the initial
  certificate-path correction; restarting disposable fixture etcd resolved it.
  The upstream CoreDNS fixture also requires `NET_BIND_SERVICE` to execute its
  capability-bearing binary, even on nonprivileged ports.
- The originally selected DNSutils image tag does not exist. Both temporary
  role probes and persistent fixtures now use `registry.k8s.io/e2e-test-images/agnhost:2.39`,
  the Kubernetes project's documented DNS-debugging image, which includes `dig`.
  Fixture preparation subsequently passed.
- First actual role convergence succeeded. Idempotence now passes with zero
  changes: pure rendering/plugin inspection explicitly report no changes, and
  Molecule's idempotence skip tag is accepted without skipping role checks.
- Read-only Molecule verification passed with zero changes, including control
  plane, CNI, artifacts, RBAC, Service/EndpointSlice/pod checks, cross-worker and
  per-replica UDP/TCP A/SRV/PTR/NXDOMAIN/headless records, pod search domains,
  and HTTP health/readiness/metrics.
- Successful candidate preflight/upgrade, explicit rollback and all four
  lifecycle check modes passed. The lifecycle runner now checks expected errors
  on both stdout and stderr and bounds failure output.
- Extended acceptance exercises cover real revision pruning, foreign production
  ownership across all mutation paths, and an actual admission-policy-induced
  recovery failure followed by explicit repair. All passed, including collisions
  involving resources present only in a rollback target.
- Fixture dependency versions: Kubernetes controller role `30.0.0+1.35.5`,
  worker role `32.0.0+1.35.5`, Cilium role `18.0.0+1.19.8`, and etcd role
  `15.0.0+3.5.26`. The initial distribution Kubernetes Python client emitted an
  upstream support warning. Final acceptance used isolated Kubernetes Python
  client `35.0.0` instead, without that compatibility warning.
- Upstream Helm rollback does not check ownership of reintroduced resources.
  Restoration now reads the target revision's manifest, requires the role marker,
  and checks target resource ownership before invoking rollback, including in
  check mode and automatic recovery.
- Effective PDB protection is checked before mutation and compared with live
  state during health checks. Offline cases cover missing/zero protection,
  mismatched selectors, acknowledged removal, and the two-replica percentage
  boundary (50% allowed, 51% rejected).
- Preflight-created probe clients now also check the existing recovery baseline
  before candidate deployment. The successful rollout exercise tests that path
  and samples UDP/TCP continuously on both workers.
- The full lifecycle suite passed, including actually pruned history, probe-only
  updates without pod restarts, all foreign-ownership paths, rollback-only
  resource collisions, forced recovery failure, cleanup, and explicit repair.
  Additional targeted execution passed continuous UDP/TCP rollout sampling,
  temporary role probe creation, atomic fresh-install failure, and reinstall.
- Guest SDK compatibility was corrected with an isolated Kubernetes client
  35.0.0 environment managed by checksum-pinned guest-local uv. The inherited
  inventory-level `ansible_become: true` overrode task/play `become: false`;
  that global override was removed so preparation escalates explicitly while
  role execution and guest-local SDK/plugin installation use the intended user.
  Forced preparation and the full acceptance suite passed under that corrected
  context, including intended non-root role execution.
- Repeated preparation exposed a template-inherited runtime-directory issue:
  the containerd dependency chmods its extraction directory to `0700`, so using
  `/tmp` broke apt signature checks on controllers/workers. The fixture now
  uses a dedicated extraction subdirectory, restores conventional `/tmp` modes,
  and aborts preparation globally on bootstrap failure to prevent a partially
  failed inventory from producing incomplete WireGuard peer configurations.

## Final acceptance

- Controller tooling: ansible-core `2.21.3`, kubernetes.core `6.5.0`,
  Molecule `26.9.0`; Helm `3.19.5` and guest-local Helm diff `3.14.0`.
  The shared controller environment and Helm plugin state were not modified.
- Final real-chart offline acceptance passed, including CLI boolean strings,
  effective PDB boundaries, and rejection of Helm hooks and retained resources.
- `ansible-lint --offline .` passed with zero failures/warnings across 54 files.
  Targeted YAML lint passed for role, example, test, and scenario YAML.
  The unrelated existing release-workflow YAML issue remains untouched.
- Nine-VM forced preparation, convergence, idempotence, read-only verification,
  and the complete lifecycle suite passed with the supported guest SDK.
  Lifecycle coverage includes preflight, authenticated DoT/TLS rejection,
  explicit rollback, automatic recovery, forced recovery failure, real history
  pruning, guarded removal/reinstall, and uninterrupted sampled UDP/TCP answers
  during the tested successful rollout. This sampling is not an availability
  guarantee for other workloads or environments.
- A separate rollback check-mode regression successfully found retained
  revision `2` with desired history limit `3`, without changing the release.
  History lookup must inspect retained revisions beyond the desired new limit;
  Helm may subsequently prune them during mutations.
- Final default convergence passed again after lifecycle testing. Idempotence
  and comprehensive verification both reported zero changes and zero failures.
  Verification also confirmed removal of candidate/client/sampler resources
  and the temporary recovery-blocking admission policy.
- `molecule destroy` succeeded. Scenario-specific Vagrant status confirmed all
  nine guests were not created; generated scenario CA artifacts were removed.
- Helm 4, dual-stack, adoption/migration, and distribution-managed DNS replacement
  remain outside the approved and tested scope.
