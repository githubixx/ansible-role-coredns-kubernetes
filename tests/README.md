<!--
Copyright (C) 2026 Robert Wimmer
SPDX-License-Identifier: GPL-3.0-or-later
-->

# Tests without a Kubernetes cluster

These tests render the real CoreDNS Helm chart and check how the role handles
configuration and lifecycle tags. They run on the machine executing Ansible and
do not require Molecule, Vagrant, or a working Kubernetes cluster.

Here, **offline means without a working cluster, not without network access**.
Helm still needs access to the configured chart repository to fetch the pinned
chart. These are not tests with a mocked chart.

## What each file does

| File | Purpose |
| --- | --- |
| [offline.yml](offline.yml) | The test suite to run. It prepares temporary output, executes the positive and negative cases, checks their results, and removes its output afterward. |
| [render.yml](render.yml) | A helper playbook that invokes the role through a static `roles:` entry. The suite uses it to render manifests, test inherited tags, and exercise invalid inputs. |
| [include.yml](include.yml) | A helper playbook that invokes the role through `include_role`. It checks that a dynamically included role is reachable when the caller selects the rendering tag. |
| [render_case.yml](render_case.yml) | A task file included by the suite for each configuration example. It renders that example and checks that the expected text is present and unwanted text is absent. It is not a standalone playbook. |

## Prerequisites

- Install ansible-core, Helm 3, the Ansible collections, and the local Python
  dependencies listed in the [role requirements](../README.md#requirements).
- Make the role available to Ansible as `githubixx.coredns` through the configured
  role search path.
- Ensure `ansible-playbook` and `helm` are available on the command search path.
  The suite launches additional `ansible-playbook` processes and passes along
  the active Ansible role and collection search paths.
- Allow access to the chart repository and permission to create temporary files.

No administrator kubeconfig is needed. The suite points both `KUBECONFIG` and
`K8S_AUTH_KUBECONFIG` at a nonexistent file in its temporary directory. Some cases
intentionally reach the role's cluster-access checks and must fail there rather
than operate on a real cluster.

## Run the suite

From the role's root directory:

```sh
ansible-playbook -i localhost, tests/offline.yml
```

The comma after `localhost` tells Ansible that this is a host list, not an
inventory filename. The playbook uses the same Python interpreter that is running
Ansible for its local modules.

The run succeeds only when the expected successful cases pass and every negative
case fails with the expected error. A failure for an unrelated reason, such as a
missing dependency, does not count as a successful negative test.

The suite creates a separate temporary directory for each run and removes it
through an `always` cleanup task, including after a test failure. Its temporary
file operations may report changes; that is not an idempotence result for a
deployed CoreDNS release.

Run the suite rather than launching the helper playbooks directly. The helpers
depend on the output directory supplied through `COREDNS_TEST_OUTPUT` and on
the tags and variables chosen by the suite.

## What is checked

- The pinned chart renders the expected DNS Service IP, two replicas, rollout
  settings, configuration checksum, cluster domain, and upstream forwarding.
- The [DNS-over-TLS](../examples/quad9-dot.yml.j2) and
  [custom-zone](../examples/custom-zones.yml.j2) examples produce the expected
  configuration.
- Explicit chart overrides take precedence over a selected values template.
- Reduced replica counts and changes to PodDisruptionBudget protection require
  the appropriate acknowledgement.
- A dynamic `include_role` works with the explicit rendering tag.
- Template check mode does not create a missing output directory.
- Conflicting or unknown lifecycle tags, skipped mandatory validation, invalid
  Service IPs, unsafe availability settings, resource-retention annotations,
  Helm hooks, and unacknowledged removal fail for the expected reason.
- Broad and inherited tags still reach normal deployment checks.
- The supported lifecycle tags appear in `--list-tags` output.

## What these tests do not prove

Rendering does not start CoreDNS, contact an upstream DNS resolver, or prove that
Kubernetes will accept and run the generated resources. These tests do not check
live DNS answers, certificate authentication, pod readiness, upgrade availability,
or rollback of an installed release.

Use the [Molecule scenario](../molecule/default/molecule.yml) for those integration
tests. The [main testing guide](../README.md#testing) explains how to run
convergence, idempotence, verification, and lifecycle exercises against its VMs.
