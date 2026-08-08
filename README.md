# Crossplane + Argo CD GitOps Pipeline — AWS S3 Provisioning

A self-service infrastructure pipeline where a `git push` provisions real AWS
infrastructure — no manual `kubectl apply`, no console clicking. Built on a
kind (Kubernetes-in-Docker) cluster running on an EC2 instance.

## Architecture

```
Engineer commits YAML
        │
        ▼
   Git repository  (XRDs, Compositions, Claims)
        │
        ▼
     Argo CD        (detects drift, auto-syncs)
        │
        ▼
Kubernetes + Crossplane   (reconciles claims into managed resources)
        │
        ▼
   AWS (S3 bucket created via provider-aws-s3)
```

The full reconciliation chain, visualized by Argo CD — from the `Bucket`
claim down to the `ProviderConfigUsage` that ties it to real AWS credentials:

![Argo CD full resource tree](screenshots/01-argocd-tree.png)

## Tech stack

| Component | Purpose |
|---|---|
| kind | Local Kubernetes cluster (Kubernetes-in-Docker) |
| Crossplane v2 | Control plane translating Kubernetes YAML into cloud API calls |
| provider-aws-s3 | Crossplane provider that talks to the AWS S3 API |
| function-patch-and-transform | Crossplane composition Function (Pipeline mode) |
| Argo CD | GitOps engine — syncs cluster state to match the git repo |
| AWS EC2 | Host running the cluster and tooling (Ubuntu 26.04, m7i-flex.large) |
| GitHub | Source of truth for all infrastructure definitions |

## Repository structure

```
crossplane-gitops-aws/
├── crossplane-config/
│   ├── provider.yaml           # installs provider-aws-s3
│   ├── function.yaml           # installs the patch-and-transform Function
│   └── provider-config.yaml    # AWS credentials config (references a Secret)
├── compositions/
│   ├── xrd-bucket.yaml         # the API: "give me a Bucket"
│   └── composition-bucket.yaml # the logic: XBucket → real S3 bucket (Pipeline mode)
├── claims/
│   └── my-first-bucket.yaml    # the actual request a developer edits
└── argocd-apps/
    ├── app-crossplane-config.yaml
    ├── app-compositions.yaml
    └── app-claims.yaml
```

Three Argo CD `Application` resources map 1:1 to the three top-level folders,
each auto-syncing (`prune: true`, `selfHeal: true`) whenever the repo changes.

## Result

All three Applications synced and healthy:

![All Argo CD applications synced](screenshots/02-apps-synced.png)

The bucket claim reconciled successfully, with a real external resource name:

![Bucket claim ready](screenshots/03-bucket-ready.png)

The actual bucket, confirmed to exist in the AWS account (not just as a
Kubernetes object):

![S3 bucket in AWS console](screenshots/04-s3-console.png)

## Errors encountered and how they were fixed

Documenting these deliberately — the debugging process is the actual skill
being demonstrated, not just getting a clean run on the first try.

### 1. CRD-before-CR race condition

**Error:**
```
Failed last sync attempt: one or more synchronization tasks are not valid:
failed to discover server resources for group version aws.upbound.io/v1beta1:
the server could not find the requested resource
```

**Cause:** Argo CD tried to apply a `ProviderConfig` custom resource before the
CRD that defines it existed. The CRD is only registered once the `Provider`
package (which the `ProviderConfig` depends on) finishes pulling its image and
starting up — a process that isn't instant.

**Fix:** Added `argocd.argoproj.io/sync-wave` annotations so the `Provider`
applies in wave `"0"` and `ProviderConfig` in wave `"1"`, plus
`argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true` on the
`ProviderConfig` so Argo CD doesn't fail validation just because the CRD isn't
visible yet at plan time.

### 2. `InjectedIdentity` not a supported credential source

**Error:**
```
ProviderConfig.aws.upbound.io "default" is invalid: spec.credentials.source:
Unsupported value: "InjectedIdentity": supported values: "None", "Secret",
"IRSA", "WebIdentity", "PodIdentity", "Upbound"
```

**Cause:** `InjectedIdentity` was assumed to auto-pick-up the EC2 instance's
IAM role, but that value doesn't exist in this provider version. `IRSA` and
`PodIdentity` are EKS-specific features not available on a plain kind cluster.

**Fix:** Switched to `source: Secret` — created an IAM user with S3
permissions, generated an access key, stored it as a Kubernetes `Secret` in
`crossplane-system`, and referenced it from `ProviderConfig`.

### 3. Composition requires Pipeline mode

**Error:**
```
Composition.apiextensions.crossplane.io "xbuckets.storage.example.org" is
invalid: spec: Invalid value: an array of pipeline steps is required in
Pipeline mode (retried 5 times).
```

**Cause:** The classic `resources:` + `patches:` array format for Compositions
was deprecated in favor of `mode: Pipeline`, which delegates patching logic to
a separate Function package rather than doing it inline.

**Fix:** Installed `function-patch-and-transform` as a Crossplane `Function`
resource, then rewrote the Composition to use `mode: Pipeline` with a single
pipeline step referencing that Function.

## Full resource graphs (bonus detail)

For anyone curious how deep the reconciliation goes, the individual
Application trees below show every resource Argo CD manages under the hood —
CRDs, ClusterRoles, the Function pipeline's ServiceAccount/Secret/Deployment,
and the Composition/CompositeResourceDefinition pairing.

**`crossplane-config`** — provider, function, and provider-config, fully synced:

![crossplane-config resource tree](screenshots/05-crossplane-config-tree.png)

**`compositions`** — XRD and Composition, with the CRDs and RBAC they generate:

![compositions resource tree](screenshots/06-compositions-tree.png)

## Key learnings

- Sync-wave ordering matters not just across separate Argo CD Applications,
  but *within* a single Application's resource list.
- Crossplane's provider/version ecosystem moves fast — credential-source enums
  and Composition modes both changed since the tutorials this was based on
  were written, which forced real troubleshooting rather than copy-paste.
- Separating `crossplane-config` / `compositions` / `claims` into distinct
  Applications cleanly maps "platform team" changes vs. "developer" changes,
  which is the actual value proposition of this pattern in production.

## Reproducing this

1. Provision an EC2 instance (Ubuntu, 2 vCPU / 8GB RAM minimum) with an
   IAM user or role that has S3 access
2. Install Docker, kubectl, kind, Helm
3. `kind create cluster`, install Crossplane and Argo CD via Helm
4. Fork/clone this repo, update `repoURL` in `argocd-apps/*.yaml` to your fork
5. `kubectl apply -f argocd-apps/` and watch Argo CD take it from there
