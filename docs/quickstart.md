# Quickstart

Install the Oxide KOG operator and watch a `Project` and an `Instance` you declare in Kubernetes appear in
the Oxide console.

## Prerequisites

- A Kubernetes cluster running **Krateo ≥ 2.5.1**, with `oasgen-provider` and `rest-dynamic-controller`
  installed (they ship with Krateo). These two provide the generic reconciliation; this chart only ships
  the per-resource OpenAPI subsets and RestDefinitions.
- Network reachability from the cluster to your **Oxide silo API** (e.g. `https://oxide.example.com`).
- The [`oxide` CLI](https://docs.oxide.computer/cli) and `kubectl`/`helm` on your workstation.
- A namespace to install into — this guide uses `krateo-system`.

## 1. Mint an Oxide API token into a Secret

Oxide authenticates with a long-lived device access token (`Authorization: Bearer <token>`). Log in once,
then capture the token into a Kubernetes Secret:

```bash
oxide auth login --host https://oxide.example.com

kubectl create secret generic oxide-api-token \
  --from-literal=token="$(oxide auth status --token)" \
  -n krateo-system
```

> The token's scope decides what you can manage. A silo-collaborator token covers projects and everything
> inside them. The fleet-scoped resources (`OxideSilo`, `OxideIpPool`, `OxideSubnetPool`) need a
> fleet-admin token.

## 2. Install the operator layer

```bash
helm upgrade --install oxide-kog ./chart -n krateo-system \
  --set oxide.apiUrl=https://oxide.example.com
```

This creates 19 `RestDefinition`s (one per resource) and the ConfigMaps holding their OpenAPI subsets.
oasgen-provider turns each RestDefinition into a CRD pair — a `<Kind>Configuration` and the resource
`<Kind>` itself. Wait for the CRDs to register:

```bash
kubectl get restdefinitions -n krateo-system
kubectl get crds | grep oxide.ogen.krateo.io
```

If you lack fleet-admin scope, disable the fleet resources:

```bash
helm upgrade --install oxide-kog ./chart -n krateo-system \
  --set oxide.apiUrl=https://oxide.example.com \
  --set restDefinitions.silo.enabled=false \
  --set restDefinitions.ippool.enabled=false \
  --set restDefinitions.subnetpool.enabled=false
```

## 3. Wire the token into the Configuration CRs

Each generated `<Kind>Configuration` references the token Secret. Apply the ready-made set:

```bash
kubectl apply -f chart/samples/00-configurations.yaml
```

## 4. Declare a project and an instance

```bash
kubectl apply -f chart/samples/10-project-and-instance.yaml
```

This declares an `OxideProject` (`demo`), an `OxideVpc`, an `OxideDisk` and an `OxideInstance` that boots
from it. Remember the conventions:

- `spec.name` is the natural key Oxide addresses the resource by; the server UUID lands in `status.id`.
- project-scoped resources take `spec.project`; VPC children take `spec.vpc`; router routes take
  `spec.router`; network interfaces take `spec.instance`.

> Edit `10-project-and-instance.yaml` first: replace the placeholder `image_id` on the disk with a real
> image UUID from your silo (`oxide image list --project demo` once the project exists, or use a
> silo image).

## 5. Verify

Watch the resources reconcile:

```bash
kubectl get oxideproject,oxidevpc,oxidedisk,oxideinstance -n krateo-system
kubectl describe oxideinstance demo-vm -n krateo-system
```

A reconciled resource reports `READY=True` and a populated `status.id`. Cross-check in Oxide:

```bash
oxide project list
oxide instance list --project demo
```

…or open the Oxide web console — the `demo` project and `demo-vm` instance are now there.

## 6. Clean up

Deleting the CRs deletes the Oxide resources:

```bash
kubectl delete -f chart/samples/10-project-and-instance.yaml
```

## Troubleshooting

- **CR stuck not-ready** — `kubectl describe` it and check the condition message. Enable verbose connector
  logs with `--set verbose=true` and read the `rest-dynamic-controller` pod logs.
- **401/403 from Oxide** — the token is missing, expired, or lacks scope for that resource. Recreate the
  `oxide-api-token` Secret (step 1) with a token of the right scope.
- **`project` required errors** — a project-scoped CR is missing `spec.project` (the project *name*).
- **No update happening** — disks, snapshots, images, internet gateways, SSH keys, certificates and silos
  are immutable in Oxide and carry no `update` verb here; change them by delete + recreate.
