# PAHP GitOps workflow — issue #1

## Repository boundaries

- `platform-infra`: Ansible owns the K3s/Cilium/Argo CD bootstrap, shared Gateway, and initial GitOps registration.
- `platform-gitops`: reviewed desired application deployment state; Argo CD Applications and immutable image digests when application CI exists.
- Each application repository: source, Dockerfile, Helm chart and namespaced HTTPRoute.

Each enabled environment gets its own namespace: `<app>-acc`, `<app>-prod`. The two environments share one K3s cluster. Neither branch is itself a namespace.

## Release contract

Owner approves and merges application code to `develop`; CI builds/tests and publishes an image digest; authorized automation updates the **approved ACC digest** in GitOps; Argo CD reconciles ACC automatically. PROD promotion requires owner-approved `main` merge and successful checks, then an approved PROD digest update. No agent may merge protected branches.

This issue does **not** implement CI or digest promotion. Do not use mutable `latest` tags.

## Safe smoke test (manual)

The initial sample is intentionally just a ConfigMap: no image pull, Pods, public ingress or Gateway changes. It tests Git read/sync without application rollout risk.

After reviewing and merging this branch into `develop`, the owner can explicitly register the sample Application on PAHP:

```bash
sudo k3s kubectl apply -f /path/to/checkout/bootstrap/applications/demo-acc.yaml
sudo k3s kubectl -n argocd get application demo-acc
```

The command expects the manifest to be present on the machine where it runs; it does not automatically appear on PAHP. Alternatively copy the file safely to the VPS. Argo CD must be able to access this public GitHub repository.

Automatic sync is intentionally **not enabled** for this first sample. In the Argo CD UI, inspect the Application and explicitly click **Sync**. Verify:

```bash
sudo k3s kubectl -n demo-acc get configmap gitops-smoke-test -o yaml
sudo k3s kubectl -n argocd get application demo-acc
```

Edit `apps/demo/acc/configmap.yaml` through a reviewed PR to `develop`, and manually sync again. The changed `data.message` verifies reconciliation. Check Argo CD reports `Synced` and `Healthy`.

## Cleanup and rollback

Delete the temporary Application **without cascading resource deletion**, then delete the test namespace explicitly:

```bash
sudo k3s kubectl -n argocd delete application demo-acc --cascade=orphan
sudo k3s kubectl delete namespace demo-acc
```

Review namespace contents before deleting. To reverse a desired-state change in a lasting application, revert the Git commit and let the approved sync policy reconcile; never edit live resources as the only rollback record.

## Explicit remaining work

- `platform-infra#46`: reproducible Ansible bootstrap registration of this GitOps repository/Application (currently manual for smoke test).
- Real application Helm chart + immutable digest promotion, ACC auto-sync, and PROD definitions require subsequent reviewed changes and tests.
- No live deployment, cluster access, idempotence, or Argo CD reconciliation has been executed by this PR.
