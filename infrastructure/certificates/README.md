# Shared PAHP TLS certificates

Argo CD Application `pahp-platform-certificates` owns the shared cert-manager Certificate resources in `kube-system`, the namespace of the shared Cilium Gateway. Do not commit private keys or generated TLS Secrets.

Add one Certificate resource per hostname and list it in `kustomization.yaml`. The `letsencrypt-prod` ClusterIssuer is provisioned separately through Ansible.

## First-time bootstrap

After merging this branch through `develop` to `main`, register `bootstrap/applications/platform-certificates.yaml` once with kubectl, then manually Sync the new Argo CD Application.

Check:

```bash
sudo k3s kubectl -n kube-system get certificate gielvangaal-dev
sudo k3s kubectl -n kube-system describe certificate gielvangaal-dev
sudo k3s kubectl -n kube-system get certificaterequests,orders,challenges
```

The ClusterIssuer uses HTTP-01 via `kube-system/pahp-http`. Public DNS A record must reach the VPS, port TCP/80 must be open, and the `kube-system` namespace must be allowed by the Gateway's allowedRoutes selector. The Gateway HTTPS listener and HTTP redirect are **separate**, later changes owned by Ansible. Do not enable HTTPS redirect before ACME validation is verified.

The Application is intentionally manually synced. Do not prune shared resources until the ownership model has been reviewed.
