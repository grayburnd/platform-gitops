## Platform GitOps

This repository bootstraps and operates the shared services required by the voting platform on Amazon EKS. It is separate from the workload repositories because it owns cluster-wide controllers, CustomResourceDefinitions, ArgoCD Projects, platform applications and the compute configuration used by team workloads.

The Terraform infrastructure layer creates the cluster foundations and AWS integrations. See the [IaC guide](https://github.com/YOUR_GITHUB_ORG/aws-eks-gitops-argocd-terraform/blob/main/IaC/README.md) and the [AWS infrastructure repository](https://github.com/YOUR_GITHUB_ORG/aws) for that boundary.

## Repository Structure

| Path | Responsibility |
|------|----------------|
| `appsets/` | ApplicationSets for platform applications, controllers and CRDs |
| `apps/` | Platform-owned Helm applications such as networking, storage and External Secrets dependencies |
| `bootstrap/` | Bootstrap charts for Fargate observability, Karpenter compute and Prometheus dependencies |
| `controllers/` | Values and source references for Argo Rollouts, AWS Load Balancer Controller, External Secrets, Karpenter, Prometheus, PostgreSQL, Redis and VPA |
| `crds/` | Third-party CRD sources including Gateway API and External Secrets |
| `projects/` | ArgoCD Projects for `backend`, `data`, `frontend` and `platform` |

Example values files are included for reference for all self-maintained Helm charts. For the third-party-maintained charts referenced under `controllers/`, use the referenced third-party Helm repository as the source of truth and fetch and use its Helm values chart.

## ArgoCD ApplicationSets

The platform ApplicationSet discovers Helm applications under `apps/*` and deploys them to the `platform-prod` namespace. The controller ApplicationSet reads `controllers/*/prod-config.yml`, combines repository values with third-party Helm sources and deploys controllers to `kube-system`. The CRD ApplicationSet reads `crds/*/prod-config.yml` and applies the referenced CRDs to `kube-system`.

Applications use automated sync with pruning and self-healing where configured. Server-side apply is enabled for large resources such as CRDs. The ArgoCD Projects constrain workload repositories to their team namespaces and provide the authorization boundary used by the frontend, backend, data and platform repositories.

## Shared Platform Services

The repository supplies the controllers and resources that support the application repositories:

- Argo Rollouts for manual-promotion blue/green deployments.
- AWS Load Balancer Controller and Gateway API resources for external HTTP routing.
- External Secrets for AWS Secrets Manager integration.
- Karpenter for team capacity and workload node provisioning.
- kube-prometheus-stack, ServiceMonitors and Grafana resources for observability.
- PostgreSQL and Redis operators for data services.
- Vertical Pod Autoscaler for resource recommendations and autoscaling support.

Karpenter configuration defines team-labelled capacity using Bottlerocket nodes, Spot capacity, supported CPU sizes and the configured availability zones. The exact limits and instance selection remain in the chart values and should be changed there rather than copied into workload repositories.

## Validation and Reconciliation

Pull requests run [`.github/workflows/ci-lint.yml`](.github/workflows/ci-lint.yml). The workflow runs Gitleaks, Helm lint and strict Kubeconform validation for `apps` and `bootstrap` charts:

```bash
helm lint -f=prod-values.yml apps/<application>
helm template apps/<application> -f prod-values.yml | kubeconform -ignore-missing-schemas -strict
```

Controller and CRD applications are reconciled by ArgoCD from the `main` branch. See the [umbrella GitOps guide](https://github.com/YOUR_GITHUB_ORG/aws-eks-gitops-argocd-terraform/blob/main/GitOps/README.md) for the complete deployment flow and the [frontend](https://github.com/YOUR_GITHUB_ORG/frontend-gitops), [backend](https://github.com/YOUR_GITHUB_ORG/backend-gitops) and [data](https://github.com/YOUR_GITHUB_ORG/data-gitops) repositories for workload-specific configuration.
