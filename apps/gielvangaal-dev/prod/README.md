# gielvangaal.dev production: phase 1

One namespace, one replica per component. The frontend and API are built by the portfolio repository's GitHub Actions; MySQL uses the official image.

## Prerequisites (before Argo CD Sync)

1. Merge this change into `platform-gitops/main`; the Argo CD Application explicitly tracks `main`.
2. Confirm both GHCR images are pullable by the K3s node; if packages are private, add an imagePullSecret to the namespace and reference it in the Pod specs.
3. Create namespace `gielvangaal-dev-prod` and the Kubernetes Secret `portfolio-db` with keys `password` and `root-password` **out of band**. Do not commit passwords or SealedSecret plaintext to Git.
4. Apply `bootstrap/applications/gielvangaal-dev-prod.yaml` to Argo CD, then manually Sync. Auto-sync is intentionally off.

## Checks

```bash
sudo k3s kubectl -n gielvangaal-dev-prod get pods,svc,pvc
sudo k3s kubectl -n gielvangaal-dev-prod logs deploy/portfolio-api --tail=80
sudo k3s kubectl -n gielvangaal-dev-prod port-forward svc/portfolio-frontend 8081:80
# In another terminal:
curl -I http://localhost:8081/
```

The frontend is not yet end-to-end functional from a browser: no Gateway/HTTPRoute is included here. A later change will expose `/` to frontend and `/api` plus `/media` to the API, all on the same hostname. The API and MySQL Services are ClusterIP only, never LoadBalancer/NodePort.

## Known considerations

- API startup deliberately recreates and seeds the MySQL database; API replicas remain at one. The DB is ephemeral in terms of its *data model*, but storage is kept on a single-node local-path PVC.
- The API must start after MySQL is usable; a CrashLoopBackOff may occur temporarily until MySQL initializes. Kubernetes retries it.
- The `mysql:8.4` tag should be pinned by digest after verifying availability.
- No public DNS or TLS changes in this phase.
