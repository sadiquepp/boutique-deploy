# boutique-deploy

The deployed state of [Online Boutique](https://github.com/sadiquepp/openshift/tree/main/test-workloads/online-boutique)
on this estate: which registry, and which exact image, per cluster and per base
set. Argo CD pulls from here. The build pipeline pushes to here.

Nothing else lives here. The application's manifests, overlays, Containerfiles
and build scripts are in
[sadiquepp/openshift](https://github.com/sadiquepp/openshift) and are referenced
as a remote kustomize base, not copied.

## Why this is a separate repository

- **Least privilege.** The pipeline needs push access to exactly one repository.
  `sadiquepp/openshift` holds Terraform, Ansible and cluster provisioning; a
  token that can rewrite the deployed image set should not also be able to
  rewrite how clusters are built.
- **The upstream repository stays a set of examples.** Its convention is
  placeholder values — `registry.example.com:8443`, `example.com`. Real
  hostnames and digests belong somewhere whose purpose is to hold them.
- **A commit per build, in a repository whose job that is.** No build noise in
  the history of the infrastructure code.
- **No loop.** The pipeline reads its own definitions from `sadiquepp/openshift`
  and writes here, so it never writes to the repository it reads itself from.

## Layout

```
hub1/hardened/kustomization.yaml    Red Hat Hardened Images -> namespace boutique-hi
hub1/ubi/kustomization.yaml         UBI 9                   -> namespace boutique-ubi
```

One directory per cluster, one per base set below it. Add a cluster by copying
`hub1/` and changing the namespaces and the registry.

Each file does three things and nothing else:

| | |
|---|---|
| `resources:` | the overlay in `sadiquepp/openshift`, as a remote base |
| `namespace:` | overrides what that overlay sets, so it needs no per-cluster copy |
| `images:` | the real registry, and a digest per service |

## The thing that is easy to get wrong

The upstream overlay **already** rewrites every image to
`registry.example.com:8443/online-boutique/<service>:v0.10.6`. A parent
kustomization's `images:` matches on the name **after** that rewrite — not on
upstream's `us-central1-docker.pkg.dev/...` name.

Key an entry on the upstream name and it matches nothing, silently. The
placeholder registry then survives into the cluster and the pods fail as
`ImagePullBackOff`, which reads like a registry or credentials problem. Both
forms were rendered to confirm which one matches.

`digest` and `newTag` are mutually exclusive: a digest replaces the tag, so the
rendered reference is `<newName>@sha256:...`.

## What the pipeline changes

Only the `digest:` lines. One per service, per base set:

```diff
   - name: registry.example.com:8443/online-boutique/frontend
     newName: registry.hub.mylab.com:8443/online-boutique/frontend
-    digest: sha256:85a80b3d…
+    digest: sha256:1f4c9e02…
```

`cache` is the exception — it is mirrored rather than built, so no pipeline run
reports a digest for it and it stays tag-pinned.

## Checking a change before Argo CD applies it

```bash
kustomize build hub1/hardened | grep -E 'image:|namespace:' | sort -u
```

Expect every image to carry `@sha256:`, every resource to be in the target
namespace, and no `Namespace` object — that one is the platform admin's, created
separately and labelled for the Argo CD instance.

`kustomize build` fetches the remote base over HTTPS from github.com, and so
does Argo CD's repo-server. On a disconnected cluster neither can, and the
answer there is to commit `kustomize build` output — rendered manifests —
instead of a remote base.
