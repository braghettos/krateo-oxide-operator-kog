<p align="center">
  <img src="https://raw.githubusercontent.com/krateoplatformops/krateo/main/docs/media/logo.svg" alt="Krateo" width="180"/>
</p>

<h1 align="center">Krateo 🧡 Oxide</h1>

> 📖 **[Quickstart](docs/quickstart.md)** — install the operator and watch a project and instance appear in the Oxide console.


# krateo-oxide-operator-kog

Krateo Operator Generator (KOG) packaging that turns **[Oxide Computer](https://oxide.computer/) Cloud**
resources into native Kubernetes custom resources — no hand-written controller, just a curated OpenAPI
subset per resource and a generic `rest-dynamic-controller`.

It builds on [`oasgen-provider`](https://github.com/krateoplatformops/oasgen-provider) and
[`rest-dynamic-controller`](https://github.com/krateoplatformops/rest-dynamic-controller): each
`RestDefinition` references a hand-curated OAS subset of the
[Oxide Region API](https://github.com/oxidecomputer/omicron/tree/main/openapi/nexus), and oasgen-provider
generates a CRD pair — a `<Kind>Configuration` (carrying the API-token reference) and the resource `<Kind>`
itself — reconciled against your Oxide silo.

## Resources

19 RestDefinitions, one per `chart/assets/<key>.yaml`. Kinds are prefixed `Oxide` to avoid crdgen
collisions with same-named lowercase body/path/query identifiers (e.g. kind `Vpc` vs the `vpc` query param).

| Kind | Oxide API | Verbs | Scope params | Token scope |
|------|-----------|-------|--------------|-------------|
| `OxideProject` | `/v1/projects` | create / get / update / delete | — | silo |
| `OxideInstance` | `/v1/instances` | create / get / update / delete | project | silo |
| `OxideDisk` | `/v1/disks` | create / get / delete | project | silo |
| `OxideSnapshot` | `/v1/snapshots` | create / get / delete | project | silo |
| `OxideImage` | `/v1/images` | create / get / delete | project | silo |
| `OxideVpc` | `/v1/vpcs` | create / get / update / delete | project | silo |
| `OxideVpcSubnet` | `/v1/vpc-subnets` | create / get / update / delete | project, vpc | silo |
| `OxideVpcRouter` | `/v1/vpc-routers` | create / get / update / delete | project, vpc | silo |
| `OxideRouterRoute` | `/v1/vpc-router-routes` | create / get / update / delete | project, vpc, router | silo |
| `OxideFloatingIp` | `/v1/floating-ips` | create / get / update / delete | project | silo |
| `OxideNetworkInterface` | `/v1/network-interfaces` | create / get / update / delete | instance, project | silo |
| `OxideAffinityGroup` | `/v1/affinity-groups` | create / get / update / delete | project | silo |
| `OxideAntiAffinityGroup` | `/v1/anti-affinity-groups` | create / get / update / delete | project | silo |
| `OxideInternetGateway` | `/v1/internet-gateways` | create / get / delete | project, vpc | silo |
| `OxideSshKey` | `/v1/me/ssh-keys` | create / get / delete | — | silo-user |
| `OxideCertificate` | `/v1/certificates` | create / get / delete | — | silo |
| `OxideSilo` | `/v1/system/silos` | create / get / delete | — | fleet |
| `OxideIpPool` | `/v1/system/ip-pools` | create / get / update / delete | — | fleet |
| `OxideSubnetPool` | `/v1/system/subnet-pools` | create / get / update / delete | — | fleet |

### How resources are addressed

Oxide addresses every resource by its **user-supplied `name`** (a `name` unique within the parent
collection). That name fills the path parameter on get/update/delete (`/v1/instances/{instance}`), so each
RestDefinition maps the path placeholder back to `spec.name` via `requestFieldMapping` — no `findby` is
needed (the natural key is known at creation time). The server-generated UUID is surfaced read-only in
`status.id`.

Resources nested under a parent are scoped by **query parameters** that oasgen-provider exposes as ordinary
spec fields: `project` (project-scoped resources), `vpc` (VPC children), `router` (router routes),
`instance` (network interfaces). Set them on the CR and rest-dynamic-controller sends them as `?project=…`.

### Verbs and immutability

Resources Oxide treats as immutable (disks, snapshots, images, internet gateways, SSH keys, certificates,
silos) carry **no `update` verb** — change them by delete + recreate. Power/attach actions
(`start`/`stop`/`reboot`, disk/IP attach-detach, IP-pool ranges, silo links) are single-fire RPCs outside
KOG's declarative create/get/update/delete model and are intentionally not modelled here.

## Auth: the Oxide API token

Oxide authenticates with a long-lived **device access token** sent as `Authorization: Bearer <token>`.
Because the token is long-lived, no rotation machinery (unlike the Keycloak KOG's short-lived admin tokens)
is required — supply it once via a Secret that each generated `<Kind>Configuration` references through
`spec.authentication.bearer.tokenRef`.

```bash
# Log in once with the Oxide CLI, then capture the token into a Secret.
oxide auth login --host https://oxide.example.com

kubectl create secret generic oxide-api-token \
  --from-literal=token="$(oxide auth status --token)" -n krateo-system
```

The token's scope determines which resources you can manage: a silo collaborator token covers projects and
everything inside them; the fleet-scoped resources (`OxideSilo`, `OxideIpPool`, `OxideSubnetPool`) require a
fleet-admin token.

## Install

```bash
# 1. The token Secret (see above).
kubectl create secret generic oxide-api-token \
  --from-literal=token="$(oxide auth status --token)" -n krateo-system

# 2. The operator layer (RestDefinitions + ConfigMaps).
helm upgrade --install oxide-kog ./chart -n krateo-system \
  --set oxide.apiUrl=https://oxide.example.com

# 3. The per-Kind Configuration CRs (token wiring) and some example resources.
kubectl apply -f chart/samples/00-configurations.yaml
kubectl apply -f chart/samples/10-project-and-instance.yaml
```

Disable resources you don't need (or lack token scope for) via values:

```bash
helm upgrade --install oxide-kog ./chart -n krateo-system \
  --set oxide.apiUrl=https://oxide.example.com \
  --set restDefinitions.silo.enabled=false \
  --set restDefinitions.ippool.enabled=false \
  --set restDefinitions.subnetpool.enabled=false
```

## What's in here

```
chart/
  Chart.yaml
  values.yaml                 # oxide.apiUrl + token Secret + per-resource toggles
  values.schema.json
  assets/                     # one curated OAS subset per resource (19 files)
    project.yaml instance.yaml disk.yaml ... subnetpool.yaml
  templates/
    _helpers.tpl
    configmaps.yaml           # embeds each asset (tpl-resolves oxide.apiUrl)
    rd-<key>.yaml             # one RestDefinition per resource (19 files)
  samples/
    00-configurations.yaml    # the <Kind>Configuration CRs (token wiring)
    10-project-and-instance.yaml
compositiondefinition.yaml    # points at oci://ghcr.io/braghettos/charts/...
examples/composition.yaml
docs/ARCHITECTURE.md
```

## Releasing

The chart is published to `oci://ghcr.io/braghettos/charts/krateo-oxide-operator-kog` by
`.github/workflows/release-chart.yaml` on a SemVer git tag matching `chart/Chart.yaml`'s `version`
(e.g. `0.1.0`). The `CompositionDefinition` `spec.chart.url`/`version` point at that artifact.

## License

Apache-2.0 — see [LICENSE](LICENSE).
