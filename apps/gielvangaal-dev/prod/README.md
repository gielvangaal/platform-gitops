# gielvangaal.dev production

This is the live PAHP portfolio deployment, managed through Argo CD. For the full release, sync and rollback workflow see [GitOps runbook](../../../docs/gitops-runbook.md).

- Namespace: `pahp-gielvangaal-dev-prod` (frontend, API, MySQL). No ACC namespace.
- Argo CD Application: `pahp-gielvangaal-dev-prod`, defined at `bootstrap/applications/gielvangaal-dev-prod.yaml`, tracks `main` and is **manually synced**.
- Portfolio source repository's GitHub Actions publish frontend and API images to GHCR. This directory pins image tags; new builds do not deploy automatically.
- Public hostname: `https://gielvangaal.dev`. Cilium Gateway API and HTTPRoutes expose frontend, API and HTTP-to-HTTPS redirect. API/MySQL Services remain internal.

## Initial setup prerequisites

1. Confirm both GHCR images are pullable by the K3s node. If private, configure a namespace-specific imagePullSecret and reference it in Pod specs.
2. Ensure namespace `pahp-gielvangaal-dev-prod` exists and create the `portfolio-db` Secret **out of band**, with keys `password` and `root-password`. Never commit passwords to Git. Namespace creation can also be handled during Application sync using `CreateNamespace=true`, but create it first when provisioning the Secret.
3. Merge the reviewed manifests into `platform-gitops/main`.
4. On the VPS, check out the reviewed `main` and apply `bootstrap/applications/gielvangaal-dev-prod.yaml` **once** to register the Application, then manually Sync in Argo CD. Changes to the bootstrap file are not auto-applied by Argo CD.

## Checks

```bash
sudo k3s kubectl -n argocd get application pahp-gielvangaal-dev-prod
sudo k3s kubectl -n pahp-gielvangaal-dev-prod get pods,svc,pvc,httproutes
sudo k3s kubectl -n pahp-gielvangaal-dev-prod logs deploy/portfolio-api --tail=80
curl -I https://gielvangaal.dev/
```

Check the actual API route and Argo CD's Synced/Healthy status as well as the public frontend.

## Known limitations

- MySQL is a single-node PVC workload; backup and restore are separate work.
- The API may temporarily restart while MySQL initializes. Inspect rollout events and logs if it does not recover.
- The API's database initialization/seeding behavior makes stateful rollback a separate concern.
- Image digests can be pinned later for stronger immutability.
- No automatic sync or GHCR image-tag promotion is configured.
