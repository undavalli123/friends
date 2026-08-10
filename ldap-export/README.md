# ldap-export

Publishes each ACM-managed cluster's LDAP connection profile and CA trust bundle
to Azure Key Vault, so a change to the bind password, LDAP URL, bind DN, search
base or CA certificate propagates with no manual step.

Two policies, no custom operator, no cron job.

| File | Runs on | What it does |
| --- | --- | --- |
| `policy-ldap-export.yaml` | every cluster labelled `brand_group=gms` | Reads the three source objects and renders two Secrets |
| `policy-ldap-hub-push.yaml` | the hub only (`local-cluster`) | Pulls those Secrets to the hub and pushes them to Key Vault |

Each file is self-contained — it carries its own `Placement` and
`PlacementBinding`, so either can be applied on its own.

---

## How it works

```
MANAGED CLUSTER  (brand_group=gms)
  ldap-sync/
    secondary-ldap-group-syncer  sync.yaml
    secondary-ldap-secret        bindPassword
    secondary-ca-config-map      ca.crt
            |
            |  policy 1 reads them, on the cluster itself
            v
  ldap-export/
    ldap-export-config   ldap-config.json
    ldap-export-ca       ca.crt
            |
  ==========|===========  cluster-proxy tunnel (the spoke dials out)
            |
            |  ExternalSecret x2 pulls, every 15m
            v
HUB  namespace <cluster>/
    ldap-export-secret-<cluster>
    ldap-export-ca-secret-<cluster>
            |
  ==========|===========  only the hub has egress to *.vault.azure.net
            |
            |  PushSecret x2 writes, every 15m
            v
AZURE  Key Vault via ClusterSecretStore azure-keyvault-gmis
    ldap-export-secret-<cluster>      application/json
    ldap-export-ca-secret-<cluster>   application/x-pem-file
```

### Why the hub does the pulling

An ACM `ConfigurationPolicy` is evaluated **on** the managed cluster and enforces
objects **there**. Hub templates resolve on the hub and are carried **down** with
the policy. Neither direction lets a managed cluster write data back to the hub.

So the upward hop is done by the one thing that can actually do it: External
Secrets Operator's `kubernetes` provider. Hub-side ESO opens a connection *to*
the spoke's API server, over cluster-proxy, and reads the Secret directly.

**ESO is installed on the hub only.** The managed clusters run no ESO — they only
run policy 1, which writes a plain Secret. Installing ESO on every cluster is the
*other* topology, where each cluster pushes to Azure itself; that is ruled out
here because the spokes have no Azure egress.

---

## Prerequisites

### Two namespaces, do not confuse them

| Namespace | Where | Holds |
| --- | --- | --- |
| `rhacm-policies` | hub | the Policies, Placements, PlacementBindings — and therefore the PlacementDecisions policy 2 looks up |
| `ldap-export` | managed cluster | the two Secrets policy 1 renders, and what the hub's SecretStore reads (`remoteNamespace`) |

They are unrelated. If you deploy the policies somewhere other than
`rhacm-policies`, change `$policyNs` in `policy-ldap-hub-push.yaml` to match —
**not** `$exportNs`. Getting this wrong is silent: the PlacementDecision lookup
returns nothing, the fan-out renders zero objects, and the policy reports
Compliant having created nothing.

On the **hub**:

- External Secrets Operator is installed.
- A `ClusterSecretStore` named `azure-keyvault-gmis` exists and can write to the
  vault. **Nothing here creates it.**

On the **managed clusters**, two ACM addons must be enabled. Both ship with ACM;
neither is ESO:

```sh
oc get managedclusteraddon -A | grep -E 'managed-serviceaccount|cluster-proxy'
```

If `managed-serviceaccount` is not enabled, policy 2 will create the
`ManagedServiceAccount` object but no token Secret ever appears, and the
`ExternalSecret` sits unauthenticated. That is the most likely first-run failure.

### ACM 2.13 or newer

Parsing `sync.yaml` needs the `fromYAML` template function, which first shipped
in ACM 2.13. On 2.12 or earlier this cannot be built as a pure policy.

---

## Install

Both files are applied to the **hub**. Each one declares the `ldap-export`
namespace as its first document, so either can be applied on its own and in
either order — there is nothing to create beforehand.

```sh
oc apply -f policy-ldap-export.yaml     # managed-cluster side
oc apply -f policy-ldap-hub-push.yaml   # hub side
```

Under Argo CD / OpenShift GitOps this works unchanged: Namespace sorts first in
Argo's kind ordering. If you would rather Argo own the namespace, delete the
Namespace document from both files and set `CreateNamespace=true` on the
Application instead.

Onboard a cluster with one label — policy 2 reads the Placement's decisions, so
this is all that is needed end to end:

```sh
oc label managedcluster <name> brand_group=gms
```

Offboard by removing the label. Both policies use
`pruneObjectBehavior: DeleteIfCreated`, so the Secrets and the hub-side objects
are removed and the Key Vault entry stops being refreshed. The Key Vault entry
itself is **not** deleted — `deletionPolicy: None` is deliberate, because Key
Vault soft-delete would reserve the name and stop a rebuilt cluster of the same
name from ever publishing again.

---

## Settings

All settings are plain variables at the top of each template. There is no
settings ConfigMap to keep in sync.

### `policy-ldap-export.yaml`

| Variable | Value | Notes |
| --- | --- | --- |
| `$srcNs` | `ldap-sync` | Where the three source objects live. Set in **both** ConfigurationPolicies |
| `$dstNs` | `ldap-export` | Namespace the rendered Secrets go into, on the managed cluster |
| `$searchFilter` | `(\|(sAMAccountName={0})(cn={0})(userPrincipalName={0})(uid={0}))` | Published verbatim. A constant rather than a label, because label values may only contain `[A-Za-z0-9._-]` and an LDAP filter cannot legally be one |

The `Placement` at the bottom of the file selects `brand_group=gms`.

### `policy-ldap-hub-push.yaml`

| Variable | Value | Notes |
| --- | --- | --- |
| `$akvStore` | `azure-keyvault-gmis` | Your existing Azure `ClusterSecretStore`. Both entries use it |
| `$proxyNs` | `multicluster-engine` | Namespace of the `cluster-proxy-addon-user` Service. Find it with `oc get svc -A \| grep cluster-proxy-addon-user` |
| `$refresh` | `15m` | Applies to both hops — see *Rotation latency* below |
| `$placement` | `ldap-export-clusters` | Must match the Placement name in `policy-ldap-export.yaml` |

Both entries go to the same store, so one Azure identity reads and writes both
the CA and the bind password; there is no separation between them. Splitting them
later means a second `ClusterSecretStore` and a second variable — a two-line
change.

---

## What is published

`ldap-export-secret-<cluster>`, content type `application/json`:

```json
{
  "friendlyName":    "celebration",
  "url":             "ex.celebration.contoso.com",
  "port":            636,
  "ssl":             true,
  "startTls":        false,
  "bindDn":          "CN=_svccbldaps,OU=Service Accounts,DC=celebration,DC=contoso,DC=com",
  "bindCredentials": "…",
  "searchBase":      "DC=celebration,DC=contoso,DC=com",
  "searchFilter":    "(|(sAMAccountName={0})(cn={0})(userPrincipalName={0})(uid={0}))"
}
```

`ldap-export-ca-secret-<cluster>`, content type `application/x-pem-file`: the PEM
bundle verbatim, and nothing else.

`port` is a number and `ssl` / `startTls` are booleans, not strings. All three are
derived from the single `url` field, so they cannot contradict each other. A URL
with no explicit port takes 636 for `ldaps://` and 389 for `ldap://`.

`searchBase` is taken from whichever schema stanza is in use — it tries
`augmentedActiveDirectory`, then `activeDirectory`, then `rfc2307`, preferring
`usersQuery.baseDN` and falling back to `groupsQuery.baseDN`.

---

## Rotation latency

ESO's `PushSecret` controller does not watch its source Secret, and the
`ExternalSecret` controller polls on its own interval. There are **two** such
hops, so worst-case propagation of a rotated bind password is `2 × $refresh`,
currently **30 minutes**.

Re-syncs are cheap: both sides read the current value first and skip the write
when nothing changed, so a short interval does not create Key Vault versions or
extra write cost.

Policy 1 is different — the config-policy-controller establishes dynamic watches
on every object a template reads, so an edit to `sync.yaml` or the bind Secret
re-renders almost immediately. The `evaluationInterval` in the file is only a
safety net.

---

## Everything fails closed

If a source object is missing, empty or malformed, or a derived value is out of
range, the policy refuses to render and reports **NonCompliant**, leaving the
last good value in Key Vault. Publishing `bindCredentials: ""` would break every
consumer of the vault while every component reported success — that failure mode
is designed out.

The conditions that stop a render:

- `sync.yaml` missing, empty, or not parseable as YAML
- no `.url`, or an empty one
- a URL with no scheme — defaulting a bare host to `ldap://` would publish
  `ssl:false` for a server that is actually LDAPS, and every consumer would
  believe it
- a scheme other than `ldap://` or `ldaps://`
- an IPv6 literal URL — splitting `[2001:db8::1]:636` on `:` would silently yield
  a wrong host and port, so it refuses rather than mis-parse
- a non-numeric or out-of-range port
- no `.bindDN`, or an empty one
- no `baseDN` under any of the three schema stanzas
- the bind Secret missing, or `bindPassword` empty
- the CA ConfigMap missing, or `ca.crt` empty or not containing a PEM
  `CERTIFICATE` block
- `$akvStore` blank or left as a placeholder (policy 2)

---

## Objects created

On each selected **managed cluster** (policy 1):

| Object | Name |
| --- | --- |
| Namespace | `ldap-export` |
| Secret | `ldap-export/ldap-export-config` — key `ldap-config.json` |
| Secret | `ldap-export/ldap-export-ca` — key `ca.crt` |

On the **hub** (policy 2), per selected cluster, in that cluster's own namespace:

| Object | Name | Role |
| --- | --- | --- |
| ManagedServiceAccount | `ldap-export-reader` | Gives the hub a token on the spoke |
| SecretStore | `<cluster>-remote` | Route to the spoke, via cluster-proxy |
| ExternalSecret ×2 | `ldap-export-secret-<cluster>`, `ldap-export-ca-secret-<cluster>` | Pull to the hub |
| PushSecret ×2 | same names | Push to Key Vault |

Six objects per cluster, all removed again when the cluster leaves the Placement.

---

## Troubleshooting

| Symptom | Cause | Where to look |
| --- | --- | --- |
| Policy 1 NonCompliant, template error names the object | A source object is missing, empty or malformed | `ldap-sync` on the spoke |
| ExternalSecret never syncs, no token Secret exists | `managed-serviceaccount` addon not enabled | hub, namespace `<cluster>` |
| ExternalSecret reports a connection error | cluster-proxy addon down, or `$proxyNs` wrong | hub, namespace `<cluster>` |
| PushSecret errors on the store reference | `azure-keyvault-gmis` missing, or lacks write access | hub |
| Vault entry stale but everything reports healthy | Expected for up to 30m — two 15m refresh hops | — |
| Policy 2 Compliant but renders nothing | No cluster carries `brand_group=gms` yet. Zero selected clusters is a legitimate steady state | `oc get placementdecision -n ldap-export` |
| Clusters are labelled but no PlacementDecision appears | No `ManagedClusterSetBinding` in `ldap-export`, or it binds a set your clusters are not in | `oc get managedclustersetbinding -n ldap-export`, `oc get managedclusterset` |
| Nothing happens at all, both policies Compliant | The hub is not self-managed, so policy 2 has no cluster to run on | `oc get managedcluster local-cluster` |

---

## Maintainer note: never write the hub delimiters literally

Before replicating a policy, the governance-policy-propagator runs a
hub-delimited template pass over the **raw text of the whole file**. It does
this to every policy, including one that only ever runs on the hub.

That pass does not know what a YAML comment is. If the opening and closing hub
delimiters appear anywhere in the file — including inside a `{{- /* ... */ -}}`
comment — it will try to execute whatever sits between them, and the policy
fails to apply with something like:

```
template-error; failed to parse the template JSON string {...}: template: tmpl:62: unexpected <.> in operand
```

So describe them in prose ("the hub delimiters") rather than writing them out.
The one legitimate use in this repo is real and intentional:

```
{{- $cluster := "{{hub .ManagedClusterName hub}}" -}}
```

in `policy-ldap-export.yaml`, which is how the cluster name is injected from the
hub. `policy-ldap-hub-push.yaml` contains none at all, by design — it runs on
the hub, so everything it reads is a local object and one template stage is
enough.

## Maintainer note: the template function set

Both policies are deliberately restricted to the template functions available in
`go-template-utils` **v7.0.x**, the release that ships with ACM 2.13. That
release still applies a strict sprig allowlist of 71 functions, and several
commonly used ones are **not** in it:

| Not available | Used here instead |
| --- | --- |
| `first`, `last` | `splitn` with `._0` / `._1` |
| `kindIs` | `hasKey` guards |
| `uniq` | `dict` + `keys` + `sortAlpha` |
| `trimPrefix` | `splitn "://" 2` |
| `toString` | `printf "%v"` |

`printf` is a Go template builtin rather than a sprig function, so the allowlist
does not affect it. `base64enc`, `base64dec`, `atoi`, `fromYAML` and `mustToJSON`
are ACM's own functions and are always present.

Newer ACM releases switched to a permissive model that allows all sprig functions
except `env` and `expandenv`, so anything written against the strict set keeps
working. The reverse is not true — a policy using `first` or `kindIs` renders
fine on a newer engine and fails on 2.13 with
`function "kindIs" not defined`. **If you edit these templates, stay inside the
v7.0.x set** unless you know every hub is on a newer release.
