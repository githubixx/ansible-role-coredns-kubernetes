<!--
Copyright (C) 2026 Robert Wimmer
SPDX-License-Identifier: GPL-3.0-or-later
-->

# CoreDNS examples

These examples show how to run the role and adapt the DNS configuration to your
cluster. They are starting points, not settings to copy unchanged into production.
See the [role README](../README.md) for the complete requirements and lifecycle
operations, and [the documented defaults](../defaults/main.yml) for configuration
options.

## What each file does

| File | Purpose |
| --- | --- |
| [deploy.yml](deploy.yml) | An Ansible playbook that installs or updates CoreDNS using this role. |
| [quad9-dot.yml.j2](quad9-dot.yml.j2) | A chart-values template that forwards queries to Quad9 over authenticated DNS-over-TLS. |
| [custom-zones.yml.j2](custom-zones.yml.j2) | A chart-values template that adds a static hostname and a locally served DNS zone. |

The `.yml.j2` files are not playbooks or complete Kubernetes manifests. The role
renders the selected template using your Ansible variables and passes the
resulting settings to the CoreDNS Helm chart.

## Before running an example

- Make the role and its required Ansible collections available to Ansible.
- Install the execution-host dependencies described in the role README,
  including Helm 3 and a compatible helm-diff plugin.
- Provide a kubeconfig with access to the intended cluster. You can set
  `coredns_kubeconfig` in the playbook or use `K8S_AUTH_KUBECONFIG`.
- Replace `10.32.0.254` in the playbook with the DNS Service IP configured in
  every kubelet's `clusterDNS`. If your cluster domain is not `cluster.local`,
  set `coredns_cluster_domain` to match the kubelets' `clusterDomain`.
- Ensure the target namespace exists and cluster networking is ready.

The role does not adopt an existing manifest-based or unrelated Helm installation.
Do not run these examples over an older DNS installation expecting an automatic
migration. Removing the old installation before deploying this role interrupts
cluster DNS; prepare and schedule that migration separately.

## Run the playbook

Run commands from the role's root directory. After adapting the playbook:

```sh
ansible-playbook examples/deploy.yml
```

Without a values template, the role configures Kubernetes DNS records, upstream
forwarding, caching, health/readiness checks, and metrics. It requests two replicas
with rollout and disruption protection.

To render the manifests for review without accessing the cluster or deploying:

```sh
ansible-playbook examples/deploy.yml --tags coredns_template
```

Rendering still requires Helm and the Ansible dependencies, plus access to the
chart repository. The output defaults to `~/coredns/template.yml` on the execution
host. Rendering does not prove that the configuration will work in your cluster.

## Choose a DNS configuration

Set `coredns_values_template` under `vars` in the example playbook. For Quad9:

```yaml
coredns_values_template: "{{ playbook_dir }}/quad9-dot.yml.j2"
```

For the static hostname and local zone example:

```yaml
coredns_values_template: "{{ playbook_dir }}/custom-zones.yml.j2"
```

`playbook_dir` is the directory containing the playbook. If you copy the playbook
elsewhere, copy the selected template alongside it or adjust the path.

These templates **replace the complete `servers` list**, including its plugins.
That is why they repeat Kubernetes DNS handling, health checks, and other plugins
rather than showing only the changed part. Other settings, such as replica count
and resource requests, remain supplied by the role unless you override them.
Selecting another template replaces the previous selection; the role does not
automatically combine both examples.

### Quad9 with DNS-over-TLS

[quad9-dot.yml.j2](quad9-dot.yml.j2) answers cluster DNS queries locally and
forwards other queries to `9.9.9.9` over TLS. It checks the upstream certificate
against `dns.quad9.net`. Allow outbound TCP/853 from CoreDNS and ensure its image
has the required trusted CA certificates.

This example uses Quad9 directly rather than `coredns_upstreams`. It does not
enable encryption between your pods and CoreDNS or fall back to plaintext if
the TLS connection fails.

### Static hostname and local zone

[custom-zones.yml.j2](custom-zones.yml.j2) demonstrates two different ways to
provide your own DNS answers:

- An imported `hosts` block maps `static.internal.test` to `192.0.2.7`.
- A separate server answers the `example.test` zone from a zone file. It includes
  `ns.example.test` at `192.0.2.53` and `answer.example.test` at `192.0.2.42`.

Replace the example names and documentation-only IP addresses with your own.
Keep the zone name, record names, filenames, and plugin paths consistent, and
increase the zone's SOA serial when changing its records. Adding records does
not create the services or machines they point to.

The chart stores both the imported snippet and zone file in its ConfigMap and
mounts them into CoreDNS. Their content is therefore included in Helm upgrades
and rollbacks; you do not need to create a separate ConfigMap manually.

Both templates retain the `pods insecure` Kubernetes plugin setting, which
generates pod-IP records without verifying that matching pods exist. Review
whether `pods verified` or `pods disabled` is more appropriate for your cluster.

## Change individual chart settings

Use `coredns_values` for small changes that do not need a separate template:

```yaml
coredns_values:
  resources:
    limits:
      memory: 256Mi
```

This changes the memory limit while retaining the role's CPU/memory requests and
other settings. Values are applied in this order: the role's built-in settings,
the selected template, then `coredns_values`. Nested mappings retain entries you
do not specify, but lists are replaced rather than appended. All overrides must
still pass the role's ownership and deployment safety checks.
