# PAHP GitOps runbook

This describes the **current, deliberately manual** deployment workflow. It is not an automated release or multi-tenant deployment framework.

## Application and namespace model

- One Kubernetes namespace **per application per environment that exists**: `pahp-<app>-<environment>`.
- Example: `pahp-gielvangaal-dev-prod` houses the portfolio frontend, API and MySQL. ACC is optional; the portfolio has no ACC environment.
- Application manifests live in `apps/<app>/<environment>/`. Argo CD Application definitions live in `bootstrap/applications/`.
- The portfolio Application `pahp-gielvangaal-dev-prod` watches `platform-gitops/main` at `apps/gielvangaal-dev/prod`. The default working branch is `develop`; merging there **does not deploy**.
- GitHub Actions in each application source repo build and publish images to GHCR. GitOps pins specific image tags in Kubernetes manifests. Argo CD reads the GitOps repository, **not application source commits or new GHCR tags**.
- Auto-sync and automated image promotion are intentionally disabled for the MVP.

Related architecture decision: [platform-infra #53](https://github.com/gielvangaal/platform-infra/issues/53).

## Release / promotion (manual)

1. In the application's own repository, build/test and publish the desired GHCR image(s). Record the immutable image tag (prefer a SHA tag) and verify the image exists and is pullable.
2. On a feature branch of `platform-gitops`, update the relevant `image:` references under `apps/<app>/<environment>/` (and any other intended manifest changes). Do **not** put secrets in Git.
3. Review the diff and validate the rendered manifests locally, for example `kubectl kustomize apps/gielvangaal-dev/prod`; check namespace, container images, routes and accidental changes.
4. Open a PR toward `develop` for review. Promote the reviewed change from `develop` into `main` through the repository's normal PR process. **Only `main` is tracked by the existing production Application.**
5. Open Argo CD (via the existing SSH/port-forward workflow) and select `pahp-gielvangaal-dev-prod`. Refresh and inspect the Git diff / OutOfSync status. Click **Sync** deliberately; do not enable auto-sync merely to release.
6. Wait for Synced/Healthy and validate workloads and the public endpoint. If an image or manifest causes an issue, use the rollback below.

## Verify a sync and reconciliation safely

The following is a **procedure, not evidence of a completed live test**.

1. Record the current Argo CD status and `main` commit; confirm portfolio is Healthy/Synced before testing.
2. Make a small, reviewed, reversible *metadata-only* change to a managed resource, such as adding a non-functional label to the frontend Deployment's metadata (not its Pod template). Do not modify replicas, images, the database, certificates or routes just to test reconciliation.
3. Merge that change into `main`. After Argo CD refresh, observe OutOfSync and the exact expected diff.
4. Manually Sync, confirm Synced/Healthy, inspect the live label, and record the commit SHA, Argo CD status, and observations in [issue #1](https://github.com/gielvangaal/platform-gitops/issues/1).
5. Remove the test label in a second reviewed PR and manually Sync again. Record the restored status.

With manual sync, detecting drift does **not** imply that Argo CD will automatically repair it.

## Verification commands

Run on the VPS (or through a securely configured kubeconfig):

```bash
sudo k3s kubectl -n argocd get applications.argoproj.io
sudo k3s kubectl -n argocd get application pahp-gielvangaal-dev-prod -o yaml
sudo k3s kubectl -n pahp-gielvangaal-dev-prod get deployments,pods,services,pvc,httproutes
sudo k3s kubectl -n pahp-gielvangaal-dev-prod rollout status deployment/portfolio-frontend
sudo k3s kubectl -n pahp-gielvangaal-dev-prod rollout status deployment/portfolio-api
curl -I https://gielvangaal.dev/
```

Check any API smoke test against a known, non-destructive route. Argo CD's UI is the source for the app-level sync/health assessment. Do not mistake a `curl` response alone for complete health.

## Rollback

1. Identify the last known-good GitOps commit and what changed (especially image tags, routing, or storage).
2. **Revert** the offending manifest commit via a reviewed PR into `main` (or restore the previously pinned image tag). Avoid force-pushing production history.
3. In Argo CD refresh the Application, inspect the revert diff, then **manually Sync**.
4. Check Synced/Healthy, rollout status and the public endpoint. Record the failure and revert SHAs.

A Git revert of Kubernetes manifests does **not** restore database data, PVC contents, schema migrations or deleted external resources. Assess those separately before rolling back stateful changes. Do not use a blind Deployment rollback when GitOps still describes the bad version.

## Adding another application (one-time manual registration)

1. Create `apps/<app>/prod/` (or an optional `acc/`) with a Kustomization and the required manifests, including namespace `pahp-<app>-<environment>`. Frontend/backend components belonging to one app share that namespace.
2. Define `bootstrap/applications/<app>-<environment>.yaml` using the existing portfolio Application as a model: unique Application name, `metadata.namespace: argocd`, `targetRevision: main`, matching source path and destination namespace, and no automated sync policy.
3. Merge the reviewed manifests and Application definition into `main`.
4. Register the new Application **once**: `sudo k3s kubectl apply -f bootstrap/applications/<app>-<environment>.yaml` on the VPS using the checked-out, reviewed `main` version. Ensure the Argo CD repository is connected.
5. Refresh and manually Sync in Argo CD, then inspect health, routes, certificates and app-specific behavior.

Creating a file under `bootstrap/applications/` does **not** by itself register it: no parent app/app-of-apps auto-registration is configured. Fully automatic registration and environment promotion are deferred under KISS/YAGNI.

## Secrets and state

The portfolio's `portfolio-db` Secret is provisioned out of band and must be present before deployment; never commit actual credentials. MySQL uses a PVC and app-specific startup behavior. Backups, secret automation, and broader disaster-recovery processes are separate backlog work, not assumptions of this runbook.
