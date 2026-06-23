# Architecture

This chart is a thin, declarative packaging layer. It contains **no controller code**: all reconciliation
is performed by Krateo's generic [`oasgen-provider`](https://github.com/krateoplatformops/oasgen-provider)
and [`rest-dynamic-controller`](https://github.com/krateoplatformops/rest-dynamic-controller), which must
already be installed in the cluster (they ship with Krateo ≥ 2.5.1).

## The pipeline

```
chart/assets/<key>.yaml         (hand-curated OAS 3.0 subset of one Oxide resource)
        │  embedded verbatim into a ConfigMap (templates/configmaps.yaml, via `tpl`)
        ▼
ConfigMap <release>-<key>        (data.<key>.yaml = the OAS, with oxide.apiUrl substituted)
        ▲  referenced by oasPath: configmap://<ns>/<release>-<key>/<key>.yaml
        │
RestDefinition <release>-<key>   (templates/rd-<key>.yaml — verbs, identifiers, field mappings)
        │  reconciled by oasgen-provider
        ▼
generated CRDs:  <Kind>            (the resource — spec fields derived from the OAS request bodies
                 <Kind>Configuration   + path/query params; status carries id)
        │  reconciled by rest-dynamic-controller
        ▼
HTTP calls to the Oxide Region API  (Authorization: Bearer <token>)
```

## Why each RestDefinition looks the way it does

- **`identifiers: [name]`** — Oxide's natural key is the user-supplied `name`, unique within the parent
  collection. It is known at creation time, so it can be used directly to address the resource.
- **`additionalStatusFields: [id]`** — the server-generated UUID is read-only; it is surfaced into
  `status.id` (it lives only in the response/view schema, never in a request body).
- **`excludedSpecFields: [<pathParam>]`** — the item path placeholder (`{instance}`, `{disk}`, …) would
  otherwise surface as a redundant spec field. It is excluded and its value is sourced from `spec.name`.
- **`requestFieldMapping: inPath:<pathParam> → spec.name`** on get/update/delete — wires the natural key
  into the path. No `findby` verb is needed because the key is not server-generated.
- **query params as spec fields** — `project`, `vpc`, `router`, `instance` are declared as operation-level
  `in: query` parameters. oasgen-provider exposes them as spec fields and rest-dynamic-controller sends them
  on every call where the operation declares them, scoping the request (`?project=…`).
- **`http`/`bearer` security scheme** — present in every asset; this is what makes oasgen-provider emit the
  `<Kind>Configuration` CRD with `spec.authentication.bearer.tokenRef`.

## Modelling choices

- **Tagged unions flattened to objects.** Oxide encodes several inputs as adjacently-tagged enums
  (`{ "type": "...", ... }`): disk backends, route destinations/targets, external IPs, address allocators.
  These are modelled as plain objects with a `type` enum plus the superset of value fields (the relevant
  ones required per `type`), so crdgen emits a clean CRD while the JSON sent to Oxide stays valid.
- **Curated subsets, not the full spec.** Each asset covers the fields that make sense to set
  declaratively. Power/attach actions and read-only telemetry are intentionally omitted.

## Regenerating the assets

The 19 assets and RestDefinition templates were generated from the Oxide Region API OpenAPI document
(`openapi/nexus/nexus-<version>.json` in `oxidecomputer/omicron`) by a catalogue-driven script. To track a
newer Oxide API version, update the curated property sets and re-emit; `appVersion` in `Chart.yaml` records
the Oxide API version the subsets were curated against.
