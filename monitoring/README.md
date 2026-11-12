# PAHP monitoring (issue #96)

Grafana + Prometheus for the single-node PAHP K3s cluster, managed by Argo CD. The chart version is pinned in `application.yaml`. This is **not yet deployed**; merging this PR alone does not automatically create the Argo CD Application.

## Initial deployment (owner action)

1. In the cluster, create the admin credential **before** syncing. Keep it out of Git / shell history. Example using interactive prompts:
   ```bash
   read -r -p 'Grafana admin user: ' GRAFANA_USER
   read -r -s -p 'Grafana admin password: ' GRAFANA_PASSWORD; printf '\n'
   sudo k3s kubectl create namespace pahp-monitoring --dry-run=client -o yaml | sudo k3s kubectl apply -f -
   sudo k3s kubectl -n pahp-monitoring create secret generic grafana-admin \
     --from-literal=admin-user="$GRAFANA_USER" \
     --from-literal=admin-password="$GRAFANA_PASSWORD"
   unset GRAFANA_USER GRAFANA_PASSWORD
   ```
   Creating this Secret manually is the MVP; future secret lifecycle can follow the existing Ansible Vault procedure.
2. After reviewing and merging the PR, run from a trusted workstation with an appropriately configured kubeconfig:
   ```bash
   sudo k3s kubectl apply -f monitoring/application.yaml
   ```
   Alternatively submit this manifest through the existing Argo CD bootstrap workflow. The example command is for running **on the VPS** from a checkout of `platform-gitops`; from a workstation use the existing kubeconfig and `kubectl`.
3. In Argo CD, manually **Sync** `pahp-monitoring`. The application deliberately has no automatic sync.
4. Validate:
   ```bash
   sudo k3s kubectl -n pahp-monitoring get pods,pvc
   sudo k3s kubectl -n pahp-monitoring get svc
   sudo k3s kubectl -n pahp-monitoring port-forward svc/pahp-monitoring-grafana 3000:80
   ```
   From the machine running the port-forward, open `http://localhost:3000`. In Grafana check **Connections > Data sources** for Prometheus and explore imported Kubernetes/node dashboards. Verify that targets such as kubelet, node-exporter and kube-state-metrics are UP in Prometheus before calling the metrics healthy.

## Scope / boundaries

- Internal `ClusterIP` only; no Ingress, HTTPRoute, DNS record, or public access in #96. Public HTTPS/SSO and linking from `admin.gielvangaal.dev` belong to #95.
- No Alertmanager or full logging/tracing system; seven days Prometheus retention, capped at 6 GB in an 8 Gi PVC. Persistent storage depends on the cluster default StorageClass (normally K3s local-path). It is **single-node local storage**, not off-node backup.
- Grafana settings and default dashboards are declarative; Grafana database persistence is disabled. User-created dashboards and local UI changes are therefore **not durable**. Save custom dashboards as provisioned definitions in Git when required.
- Prometheus metrics cover host/node and Kubernetes resources. Per-website HTTP status codes, requests, latency and external uptime need explicit application/gateway instrumentation or a blackbox exporter, which are **follow-up work** if unavailable.
- The chart installs CRDs/cluster-scoped monitoring components, so review its impact before the first sync. K3s/Cilium may not expose every stock control-plane scrape target.
- Existing grafana-admin Secret is a deployment prerequisite; never commit credentials or copy the Secret into source control.

## Rollback

Remove the Argo CD Application after approval and then delete its monitoring resources/namespace if desired. **Deleting the PVC permanently removes Prometheus history**. Do not delete the Secret/PVC as part of ordinary chart rollbacks. Reverting the Git commit does not by itself uninstall the live Helm release.
