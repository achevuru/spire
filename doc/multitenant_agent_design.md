# Multi-tenant SPIRE Agent design approach

This document describes an incremental approach for making a SPIRE Agent service
workloads from multiple tenant trust domains on the same node. It is intended as
a design starting point for deployments where each tenant has its own SPIRE
Server, trust domain, registration entries, and trust bundle, but workloads from
those tenants are co-scheduled on shared nodes.

## Problem statement

SPIRE Agent configuration is currently centered on a single server connection and
a single local trust domain. In multi-tenant clusters this usually means running
one SPIRE Agent per tenant on every shared node. Sidecars, node proxies, or mesh
components such as ztunnel then need tenant-aware logic to select the right
Workload API endpoint for each local workload.

The goal is to allow one node-local SPIRE Agent process to:

- maintain independent upstream connections to multiple SPIRE Server deployments;
- attest the node independently for each tenant trust domain;
- route each Workload API request to the tenant that owns the caller workload;
- return SVIDs, JWT-SVIDs, and bundles only for the selected tenant context; and
- preserve SPIFFE trust-domain isolation and SPIRE's least-privilege security
  model.

## Non-goals

The following are out of scope for the initial design:

- issuing one SVID that belongs to multiple trust domains;
- merging tenant registration APIs, datastores, or policy administration;
- allowing one tenant SPIRE Server to authorize workloads in another tenant trust
  domain without federation; and
- changing the SPIFFE Workload API contract for existing clients.

## Proposed model

Introduce a first-class **tenant** abstraction inside SPIRE Agent. A tenant is an
isolated agent runtime context keyed by a stable local name and associated with
exactly one SPIFFE trust domain.

Each tenant context owns:

- the trust domain and server address for that tenant;
- the bootstrap trust bundle configuration for that tenant;
- node-attestation state and the node SVID for that tenant;
- the server stream used to synchronize authorized entries and bundles;
- tenant-scoped X.509-SVID and JWT-SVID caches;
- tenant-scoped SDS resources, if SDS is enabled; and
- tenant-specific health, telemetry, and status fields.

The outer SPIRE Agent process owns shared host integrations such as the Workload
API listener, workload attestors, key manager, storage backend, telemetry server,
and process lifecycle. The tenant manager coordinates the per-tenant contexts.

## Configuration shape

A backwards-compatible configuration can keep the existing single-tenant fields
and add a `tenants` map for multi-tenant mode. For example:

```hcl
agent {
  socket_path = "/run/spire/agent/api.sock"

  tenants = {
    tenant_a = {
      trust_domain = "tenant-a.example"
      server_address = "spire-server.tenant-a.svc"
      server_port = "8081"
      trust_bundle_path = "/run/spire/bootstrap/tenant-a.pem"
    }

    tenant_b = {
      trust_domain = "tenant-b.example"
      server_address = "spire-server.tenant-b.svc"
      server_port = "8081"
      trust_bundle_path = "/run/spire/bootstrap/tenant-b.pem"
    }
  }

  tenant_resolver {
    plugin_cmd = "k8s_tenant"
    plugin_data {
      default_tenant = "tenant_a"
      namespace_annotation = "spire.spiffe.io/trust-domain"
      pod_annotation = "spire.spiffe.io/trust-domain"
    }
  }
}
```

If `tenants` is omitted, SPIRE Agent behaves exactly as it does today. If
`tenants` is set, single-tenant fields such as `trust_domain`, `server_address`,
and `trust_bundle_path` should either be rejected or treated as a shorthand for a
single default tenant, depending on the compatibility policy selected by the
maintainers.

## Tenant resolution

The key design requirement is choosing a tenant before authorizing a workload
request. The agent needs a tenant resolver that maps an attested workload to a
local tenant key. The resolver should run after workload attestation produces
selectors and before the agent queries tenant-scoped authorized entries.

A Kubernetes implementation could map a pod to a tenant by using one of these
ordered mechanisms:

1. a pod annotation such as `spire.spiffe.io/trust-domain`;
2. a namespace annotation or label;
3. an admission-controller-populated workload identity hint; or
4. an operator-defined default tenant.

The resolver output should include both the tenant key and the evidence used to
make the decision so that audit logs and authorization failures are explainable.
Resolvers must fail closed. If a workload cannot be mapped to exactly one tenant,
the agent should deny the request rather than returning SVIDs from an arbitrary
tenant.

## Workload API behavior

The public Workload API socket can remain single and tenant-neutral. Existing
clients connect to the same socket. Internally, each request flows through this
pipeline:

1. identify the caller by peer PID/UID/GID or platform-specific peer tracking;
2. run all configured workload attestors and collect selectors;
3. resolve the tenant from selectors and platform metadata;
4. dispatch the request to the tenant context;
5. evaluate only that tenant's authorized entries;
6. mint or retrieve tenant-scoped SVIDs; and
7. return only that tenant's bundles and federated bundles.

For compatibility with clients that can supply richer context, an optional
future enhancement could support a tenant hint in a SPIRE-specific extension.
Such a hint must be treated only as an optimization: the resolver must verify the
hint against authoritative local workload metadata.

## Server connection and cache isolation

The tenant manager should run one independent synchronization loop per tenant.
Each loop connects to that tenant's SPIRE Server, performs node attestation using
that tenant's node attestor configuration, receives entries and bundles for that
trust domain, and updates tenant-local caches.

Isolation requirements:

- cache keys must include the tenant key or trust domain;
- an entry synchronized from one tenant must never satisfy a request for another
  tenant;
- JWT-SVID and X.509-SVID LRU accounting should be per tenant, with optional
  process-wide limits as a safety valve;
- storage records should be namespaced by tenant to prevent accidental reuse of
  node SVIDs, bundle state, or workload keys; and
- status and health APIs should report each tenant independently.

## Security considerations

A multi-tenant agent collapses several node-local agent processes into one
security boundary, so the design must make tenant isolation explicit.

Important controls include:

- fail-closed tenant resolution with clear ambiguity errors;
- strict validation that the resolved tenant trust domain matches the SPIFFE IDs
  returned to the workload;
- tenant-scoped authorization checks before any SVID is minted or served;
- independent upstream mTLS sessions and bundle validation per tenant;
- audit logs that include tenant key, trust domain, workload selectors, and the
  selected registration entry;
- defensive limits so one tenant cannot exhaust shared agent memory, CPU, or
  connection capacity; and
- a conformance test suite that proves cross-tenant selector collisions do not
  leak identities.

## SDS and ztunnel integration

The main integration benefit is that node-local consumers such as ztunnel can use
a single agent endpoint. ztunnel should not have to maintain a socket per tenant.
Instead, the agent resolves the tenant from the target workload and returns SDS
resources from the matching tenant context.

For SDS resource names that already encode a SPIFFE ID, the trust domain in the
resource name can be used as a consistency check after tenant resolution. For
resource names that do not encode a trust domain, the agent must rely on the
resolver and should reject ambiguous requests.

## Migration plan

A safe rollout can be phased:

1. **Internal refactor**: factor the existing single-tenant agent state into a
   `TenantContext` while keeping one configured tenant.
2. **Configuration preview**: add parsing and validation for a `tenants` map, but
   initially gate it behind an experimental flag.
3. **Tenant resolver plugin API**: introduce a minimal resolver interface and a
   Kubernetes resolver implementation.
4. **Per-tenant synchronization**: run independent server streams, cache updates,
   and bundle management per tenant.
5. **Workload API routing**: dispatch Workload API and SDS requests through the
   resolver into the selected tenant context.
6. **Operational hardening**: add tenant labels to telemetry, health, logs, and
   debug endpoints; add resource limits and fault-injection tests.
7. **Compatibility graduation**: remove the experimental gate after scale,
   failure-mode, and security testing.

## Open questions

The following decisions should be resolved before implementation:

- Should each tenant be allowed to use a different node attestor plugin instance,
  or should all tenants share one node attestor type with tenant-specific
  configuration?
- Should key manager and storage plugins be tenant-aware, or should the agent
  namespace all keys and records before calling existing plugins?
- How should tenant mappings be administered outside Kubernetes?
- What process-wide and per-tenant cache defaults are safe for large clusters?
- Should the Workload API expose tenant status in a new admin endpoint, or should
  status remain available only through existing agent operational endpoints?
- What compatibility guarantees are required for existing SDS resource names?

## Recommended first implementation target

The smallest useful implementation is a Kubernetes-focused experimental feature:

- one SPIRE Agent process and one public Workload API socket per node;
- multiple tenant entries in agent configuration;
- one upstream SPIRE Server connection per tenant;
- a Kubernetes tenant resolver based on pod or namespace annotations;
- strict per-tenant cache and storage namespaces; and
- workload X.509-SVID, JWT-SVID, bundle, and SDS requests routed by resolved
  tenant.

This target removes the need to run one agent and one ztunnel-to-agent
integration per tenant while preserving the existing SPIRE Server trust-domain
model.
