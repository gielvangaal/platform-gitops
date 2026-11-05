# PAHP platform-gitops

Declarative application and infrastructure manifests reconciled by Argo CD. Production Applications track `main`; changes are reviewed before promotion, then **manually synced**.

- [GitOps runbook: release, sync, rollback, new applications](docs/gitops-runbook.md)
- [Portfolio production deployment](apps/gielvangaal-dev/prod/README.md)
- [Argo CD Application definitions](bootstrap/applications/)
- [Certificate manifests](infrastructure/certificates/)

One namespace per deployed application/environment (`pahp-<app>-<environment>`); ACC is optional. Application repositories build/publish container images; this GitOps repository chooses what is deployed.
