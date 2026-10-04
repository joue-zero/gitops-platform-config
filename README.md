# gitops-platform-config

Desired state of the gitops-platform Kubernetes environments. [Argo CD](https://argo-cd.readthedocs.io/) watches this repository and makes the cluster match it, so a change reaches the cluster by being merged here.

Application code, the Helm chart and the Terraform infrastructure live in [`gitops-platform`](https://github.com/joue-zero/gitops-platform).

## How it works

```text
gitops-platform                                     gitops-platform-config (this repo)
  build image -> ECR (immutable SHA tag)              envs/dev/values.yaml   image.tag
  Terraform: EKS + Argo CD + root Application ----->  apps/*.yaml            what to deploy
                                          Argo CD renders helm/app with the env values and syncs it
```

1. Terraform installs Argo CD right after the cluster, with one root Application pointing at [`apps/`](apps/).
2. The root Application deploys every manifest in `apps/` (app-of-apps), so a new workload is a pull request, not a Terraform change.
3. Each Application renders the chart from the application repository with the values in [`envs/`](envs/). `prune` removes what leaves Git and `selfHeal` reverts manual changes.

The dev environment is destroyed every night. A recreated cluster gets Argo CD from Terraform and everything else from this repository, with no manual bootstrap.

## Layout

| Path | Purpose |
|---|---|
| [`apps/project.yaml`](apps/project.yaml) | AppProject: allowed source repositories, destination namespace, and cluster-scoped resources (Namespace only). |
| [`apps/gitops-app-dev.yaml`](apps/gitops-app-dev.yaml) | Dev Application. Chart from the application repository, values from this one; enforces the `restricted` Pod Security profile. |
| [`apps/platform-addons-project.yaml`](apps/platform-addons-project.yaml) | AppProject for upstream add-ons, which need the cluster-scoped RBAC the app project deliberately lacks. |
| [`apps/metrics-server.yaml`](apps/metrics-server.yaml) | metrics-server from its upstream chart, so the HPA can read CPU usage. |
| [`envs/dev/values.yaml`](envs/dev/values.yaml) | Dev overrides of the chart defaults: image tag, ALB Ingress and the HPA. |
| [`.github/workflows/validate.yml`](.github/workflows/validate.yml) | CI: validates the manifests against the Argo CD CRD schemas and renders each environment against the real chart. |

## Release and rollback

A release is a commit that sets `image.tag` in `envs/<env>/values.yaml` to the git SHA of an image already in ECR. The `update-gitops` job in the application repository's CI makes that commit after every image push (it needs the `GITOPS_CONFIG_TOKEN` secret there). A rollback is `git revert` of that commit.

## Using Argo CD

The API server is not exposed publicly:

```bash
aws eks update-kubeconfig --name gitops-platform-eks --region eu-central-1
kubectl -n argocd port-forward svc/argocd-server 8080:443   # https://localhost:8080, user admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

Your IAM identity needs a cluster access entry: set the `EKS_ADMIN_PRINCIPALS` repository variable of the application repository (CI-created clusters) or `eks_admin_principals` in `terraform.tfvars` (local applies).

## Adding workloads and environments

- Workload: add an Application manifest to `apps/`.
- Environment: add `envs/<env>/values.yaml` and an Application pointing at it, with its own namespace. Pin the chart `targetRevision` to a tag for production, and switch to an `ApplicationSet` once several environments exist.

## Known limitations

- The dev Application tracks the chart on `main`, so chart changes reach dev as soon as they merge.
- Destroy deletes the root Application first, which cascades to the workloads. This is safe while they create only in-cluster resources. Once an Ingress creates a load balancer, its controller must still be running during the cascade, or the load balancer is orphaned and blocks VPC deletion.
