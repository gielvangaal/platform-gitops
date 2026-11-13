# PAHP monitoring (#96)

Grafana and Prometheus are deployed via Argo CD with the pinned
`kube-prometheus-stack` chart. The Application definition is maintained at
`bootstrap/applications/monitoring.yaml`, along with the existing Portfolio,
SoulSearch and certificate Applications.

## Deployment

1. Provision `grafana-admin` using the Ansible Vault-backed
   `playbooks/kubernetes-secrets.yml --tags grafana`.
2. Ensure the `pahp-root` Argo CD Application was bootstrapped **once**
   by Ansible and has synced `bootstrap/applications/`.
3. Open the `pahp-monitoring` Application in Argo CD and **manually Sync**.
4. Check the monitoring namespace:
   ```bash
   sudo k3s kubectl -n pahp-monitoring get pods,pvc
   sudo k3s kubectl -n pahp-monitoring get svc
   sudo k3s kubectl -n pahp-monitoring port-forward svc/pahp-monitoring-grafana 3000:80
   ```
5. Open Grafana at `http://localhost:3000` from the port-forward machine.
   Inspect the Prometheus datasource and verify that node-exporter,
   kubelet and kube-state-metrics targets are available.

## Updating

Edit `bootstrap/applications/monitoring.yaml` and merge into `main`.
The root Application reconciles the child Application definition automatically.
**Manual sync of `pahp-monitoring`** then applies the Helm values. No Ansible
run is required for normal chart configuration changes.

## Scope and limitations

Monitoring remains internal (ClusterIP only), with a 7-day / 6GB retention
target and an 8Gi local-path PVC for Prometheus. Grafana persistence is
disabled; any UI-created dashboards are ephemeral. The admin password is
held in Ansible Vault and a namespaced Kubernetes Secret, not Git. The
Grafana memory limit is 768Mi after OOMKilled restarts on the 256Mi limit.

No public HTTPRoute, Alertmanager, central logging or off-node Prometheus
backups are included. Exposing and authenticating monitoring belongs to
the separate admin dashboard issue.

## Rollback

Revert and merge the Application configuration, then manually sync it in Argo CD.
Do not delete Secrets or PVCs when rolling back monitoring. The root
Application has pruning disabled: deleting a child manifest does not
automatically delete the workload or Application object.
