# rancher-rke2 (Crossplane project)

Manages downstream "custom" RKE2 clusters on Rancher declaratively, from
inside the RKE2 local/management cluster, instead of via the Terraform
`rancher2` provider.

## Why provider-kubernetes instead of provider-rancher2

Rancher already models cluster provisioning as native Kubernetes CRDs on the
local cluster:

- `provisioning.cattle.io/v1 Cluster` - the downstream cluster's desired
  config (Kubernetes version, CNI, kubelet/apiserver args, upgrade
  concurrency, ...).
- `management.cattle.io/v3 ClusterRegistrationToken` - generates the node
  join command for "custom" clusters (the same curl/`system-agent-install.sh`
  command your Terraform flow currently surfaces via registration commands).

Since Crossplane runs in that same cluster, `provider-kubernetes`'s generic
`Object` managed resource can create/update/delete these directly with
`InjectedIdentity` credentials (its own ServiceAccount, no Rancher API token
needed). `provider-rancher2` would work too, but it goes back out over
Rancher's REST API and needs its own bearer token - an extra credential to
manage for no benefit when you're already inside the cluster Rancher runs on.

## Two layers, same underlying objects

- **Option 1 - direct CRs**: [bootstrap/option1-plain-objects/rke2-cluster-plain.yaml](bootstrap/option1-plain-objects/rke2-cluster-plain.yaml)
  applies the `provisioning.cattle.io Cluster` and
  `management.cattle.io ClusterRegistrationToken` as plain `Object`
  resources. This is the direct equivalent of what your Terraform module
  does today, just reconciled continuously by Crossplane instead of run on
  `terraform apply`.
- **Option 2 - the `XRke2Cluster` abstraction**: [apis/xrke2clusters/definition.yaml](apis/xrke2clusters/definition.yaml)
  (already scaffolded) + [apis/xrke2clusters/composition.yaml](apis/xrke2clusters/composition.yaml)
  (new) compose the *same* two objects from one higher-level claim - see
  [examples/clusters/rke2cluster.yaml](examples/clusters/rke2cluster.yaml).
  A consumer applies one `XRke2Cluster`; the Composition generates the
  Rancher objects underneath it. This is your self-service / GitOps layer.

Read the Composition before the plain-objects example if you want to see how
one becomes the other - the `patches` block is a field-by-field map from
`XRke2Cluster.spec.parameters.*` to the manifests in option 1.

`bootstrap/` (rather than `examples/`) holds the plain-objects demo, the
ProviderConfig, and RBAC: this project's `examples/` directory is reserved
for instances of the project's own XRD types (that's what `crossplane
composition render` and the project build step validate against) - dropping
arbitrary manifests in there breaks the build, which is how this got found.

## Setup

1. Install Crossplane on the RKE2 local/management cluster (if not already).
2. Apply this project's dependencies (already declared in
   `crossplane-project.yaml`): `provider-kubernetes` and
   `function-patch-and-transform`. If you're using the `up` CLI:
   `up project run` / `up project build && up project push`, or apply the
   `Provider`/`Function` package manifests directly if you're not using
   `up`.
3. Grant `provider-kubernetes` RBAC on the Rancher CRDs:
   `kubectl apply -f bootstrap/rbac/provider-kubernetes-rancher-rbac.yaml`
   (fix the ServiceAccount name first - see the comment in that file).
4. `kubectl apply -f bootstrap/providerconfigs/kubernetes-local.yaml`
5. Apply the XRD and Composition:
   `kubectl apply -f apis/xrke2clusters/definition.yaml -f apis/xrke2clusters/composition.yaml`
6. Try it: `kubectl apply -f examples/clusters/rke2cluster.yaml`, then
   `kubectl get xrke2cluster gpu-pool-01 -o yaml` and watch
   `status.mgmtClusterName` and `status.nodeRegistrationCommand` populate as
   Rancher reconciles the underlying objects. This takes two Crossplane
   reconciles, not one: the registration token can't be created until
   Rancher assigns the Cluster a management-cluster id, and that id has to
   round-trip through the XR's own status before the token resource's patch
   can read it - confirmed with `crossplane composition render` using a
   mocked "already reconciled" Cluster object as `--observed-resources`.
   Expect a short delay, not an error, if `nodeRegistrationCommand` is empty
   right after apply.

## Getting nodes to actually join

`status.nodeRegistrationCommand` on the `XRke2Cluster` gives you the same
`curl ... | sudo sh -s - --server ... --token ... --ca-checksum ...` command
Rancher's UI shows you, or that your current Terraform output surfaces.
**Neither Terraform nor Crossplane runs this on the target node for you** -
that's inherent to how Rancher "custom" clusters work, not a gap specific to
either tool. You still need something to execute it on each host:

- Pull the command from the XR's status and hand it to whatever already
  provisions your nodes (Ansible, cloud-init, a bootstrap script).
- Or keep the remote-exec step in Terraform and only migrate the
  cluster/token half to Crossplane.

## Known gap

`spec.parameters.nodeGroups` (per-nodegroup kubelet arg overrides matched by
label) isn't wired into the Composition yet - see the comment in
`composition.yaml`. Rancher expects those as
`spec.rkeConfig.machineSelectorConfig[]` entries
(`{machineLabelSelector, config}`), which is a reshape
`function-patch-and-transform` can't do with plain field patches. Next step
if you need it: a small custom composition function (Go, Python, or KCL -
the `schemas/` scaffold here already has generated types for all three) that
maps `nodeGroups[]` to `machineSelectorConfig[]`.

## Before you rely on this

The exact field names under `spec.rkeConfig` (`kubelet-arg`,
`kube-apiserver-arg`, `disable-kube-proxy`, ...) come from RKE2's own
`config.yaml` schema, which Rancher's provisioning API passes through
directly - it has shifted across Rancher versions. Confirm against your
actual install before trusting this in production:

```
kubectl explain cluster.provisioning.cattle.io.spec.rkeConfig --recursive
kubectl get cluster.provisioning.cattle.io <name> -n fleet-default -o yaml
```
