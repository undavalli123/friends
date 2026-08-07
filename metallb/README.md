# MetalLB — Red Hat ACM policies

Installs and configures MetalLB on managed clusters via Red Hat Advanced
Cluster Management, with the address pool derived per cluster instead of
hardcoded.

## Files

| File | Contents |
| --- | --- |
| `policy-metallb-operator.yaml` | `policy-metallb-operator` — Namespace, OperatorGroup, Subscription, CSV readiness gate, `MetalLB` CR |
| `policy-metallb-config.yaml` | `policy-metallb-config` — `IPAddressPool` + `L2Advertisement` |
| `policyset-metallb.yaml` | `PolicySet`, `Placement`, `PlacementBinding` |

Everything lives on the hub in the `rhacm-policies` namespace.

## Apply

```sh
oc apply -f metallb/policy-metallb-operator.yaml
oc apply -f metallb/policy-metallb-config.yaml
oc apply -f metallb/policyset-metallb.yaml
```

## Targeting

The `Placement` selects managed clusters labelled `brand_group=gms`:

```sh
oc label managedcluster <cluster-name> brand_group=gms
```

## How the address pool is derived

The pool is not hardcoded. Every cluster uses the same `x.x.x.192/26` machine
network with a different third octet, so the `IPAddressPool` uses an ACM
managed-cluster template that resolves on each spoke at evaluation time:

1. `lookup` the `Infrastructure/cluster` resource on the managed cluster.
2. Walk `.spec.platformSpec.vsphere.machineNetworks` and keep the entry ending
   in `/26`. The `/32` entries in that list are the API and ingress VIPs and
   must be skipped — filtering on the suffix rather than taking index `0`
   means the template is not affected if the list is reordered.
3. Strip the mask and the host octet, keeping the first three octets.
4. Rebuild the range as `<a>.<b>.<c>.213-<a>.<b>.<c>.219` — seven addresses
   carved out of the `/26`.

For a cluster whose `Infrastructure` reads:

```yaml
spec:
  platformSpec:
    type: VSphere
    vsphere:
      apiServerInternalIPs:
        - 10.106.209.194
      ingressIPs:
        - 10.106.209.195
      machineNetworks:
        - 10.106.209.192/26
        - 10.106.209.194/32
        - 10.106.209.195/32
```

the resolved pool is:

```yaml
spec:
  autoAssign: true
  addresses:
    - 10.106.209.213-10.106.209.219
```

To change which addresses are carved out, edit the two literals in the
`printf` at the end of the template in `policy-metallb-config.yaml`.

## Ordering

MetalLB's CRDs do not exist until the operator has installed, so the objects
are gated rather than applied all at once:

- `metallb-instance` (the `MetalLB` CR) has an `extraDependencies` on the
  `metallb-operator-status` ConfigurationPolicy, which is `inform` and simply
  watches for the `MetalLB Operator` CSV to reach `phase: Succeeded`.
- `policy-metallb-config` has a `dependencies` entry on
  `policy-metallb-operator` being `Compliant`, so the `IPAddressPool` and
  `L2Advertisement` are only created once the operator is up.

Both policies are `remediationAction: enforce`, so ACM creates and continually
corrects these objects. Switch to `inform` for a dry run — the policies will
then report compliance without changing anything.

## Notes

- The `lookup` in the template runs as the ACM `config-policy-controller`
  service account on the managed cluster, which needs read access to
  `config.openshift.io/v1 Infrastructure`. It has this by default; if the
  policy reports a template error mentioning `Infrastructure`, check that the
  addon's RBAC has not been narrowed.
- If the lookup returns nothing (non-vSphere platform, or no `/26` in
  `machineNetworks`), the template resolves to an invalid range and the policy
  goes `NonCompliant` with the error rather than silently creating a bad pool.
- `autoAssign: true` means any `LoadBalancer` Service without a pool
  annotation draws from these seven addresses.
- The `L2Advertisement` sets no `interfaces` or `nodeSelectors`, so the pool is
  advertised from every node and MetalLB picks the interface itself.
