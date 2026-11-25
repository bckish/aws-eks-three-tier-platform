# AWS EKS three-tier platform

Portfolio lab maintained in [bckish/aws-eks-three-tier-platform](https://github.com/bckish/aws-eks-three-tier-platform). The implementation is derived from upstream work; this repository's documentation and validation changes do not imply an independently verified cloud deployment.

React frontend, Node/Express API and MongoDB with Jenkins-based image delivery, Kubernetes manifests, and a Terraform-defined Jenkins host.

![Upstream architecture illustration](assets/Three-Tier.gif)

## Repository map

- [Application-Code](Application-Code/): frontend and backend.
- [Jenkins-Pipeline-Code](Jenkins-Pipeline-Code/): separate frontend and backend pipeline examples.
- [Jenkins-Server-TF](Jenkins-Server-TF/): AWS infrastructure for the Jenkins server.
- [Kubernetes-Manifests-file](Kubernetes-Manifests-file/): application, database and ingress manifests.

## Reproduce and validate

```bash
git clone https://github.com/bckish/aws-eks-three-tier-platform.git
cd aws-eks-three-tier-platform
node --check Application-Code/backend/index.js
terraform -chdir=Jenkins-Server-TF init -backend=false
terraform -chdir=Jenkins-Server-TF validate
```

No AWS credentials are needed for syntax validation; provider downloads require internet access. For actual deployment, configure your own backend, account, ECR repositories, images, credentials, Kubernetes cluster and ingress domain. The Jenkins jobs require configured tools and credentials; the default build checks out the current SCM revision, scans the image, and does not publish. Enable PUBLISH_IMAGE only on an approved run. Update GitOps manifests via a separate reviewed PR. Do not reuse upstream infrastructure or secrets. Cloud provisioning incurs cost and requires explicit approval.

The backend exposes /healthz, /ready and /started. The backend CRUD smoke test and Vite production build are runnable with the committed npm lockfiles. The original [walkthrough](https://amanpathakdevops.medium.com/advanced-end-to-end-devsecops-kubernetes-three-tier-project-using-aws-eks-argocd-prometheus-fbbfdb956d1a) explains the intended full deployment.

## Attribution and license

Original implementation by Aman Pathak. See [UPSTREAM.md](UPSTREAM.md) for the source repository and [LICENSE](LICENSE) (Apache-2.0). The upstream license and copyright notices are unchanged.
## Local application demo (no cloud or database required)

Use Node 22.12+ (or newer compatible Node). Run the backend with an explicit temporary in-memory store; task data is lost when the process exits. Set MONGO_CONN_STR instead of LOCAL_MEMORY for a real local MongoDB connection.

```bash
cd Application-Code/backend
npm ci
npm test
LOCAL_MEMORY=true npm start
```

In a second terminal, from the repository root:

```bash
cd Application-Code/frontend
npm ci
npm start
```

The UI defaults to http://127.0.0.1:3500/api/tasks in local mode. Set REACT_APP_BACKEND_URL when deploying it elsewhere. No cloud deployment is performed.

## Verified local Kubernetes and Argo CD results

Test date: 25 November 2026. Environment: Docker Desktop on Apple Silicon, Kind v0.30.0, Kubernetes v1.34.3. Local images were imported into the isolated cluster; no images were pushed to a registry and no paid cloud resources were created.

Tested code: `90d6c7d` (merged main). The local test used the actual frontend/backend Docker builds (`bckish-local/aws-task-api:runtime` and `bckish-local/aws-task-ui:runtime`), MongoDB 7, and a Kind-local persistent volume. The frontend responded over HTTP, the API readiness probe passed, and task create/read/update/delete passed against real MongoDB both before and after reconciliation.

Argo CD chart 5.46.0 (Argo CD v2.8.3) reconciled a disposable local Git repository, not AWS or a remote GitOps repository. The initial Git revision `dcc5f41` deployed one ready API replica; the next revision `2e8ab9e` updated a proof ConfigMap and scaled the API to two ready replicas. The Application reported **Synced / Healthy**. Manually scaling the API back to one replica was corrected automatically to two.

The [local Kubernetes fixture](local-k8s/) uses local image tags, local MongoDB credentials, a dynamic PVC and no cloud ingress. Build and load the images before applying it to a disposable cluster:

```bash
docker build -t bckish-local/aws-task-api:runtime Application-Code/backend
docker build -t bckish-local/aws-task-ui:runtime Application-Code/frontend
docker pull mongo:7
kind load docker-image --name YOUR_LOCAL_KIND bckish-local/aws-task-api:runtime bckish-local/aws-task-ui:runtime mongo:7
kubectl --kubeconfig /path/to/local-kind.kubeconfig create namespace argo-task
kubectl --kubeconfig /path/to/local-kind.kubeconfig apply -k local-k8s
```

On Apple Silicon, if multi-platform image import fails, use `docker image save --platform linux/arm64` followed by `kind load image-archive`. Forward the API and UI services to localhost ports 23500 and 23300 respectively for the included local frontend configuration. The genuinely deployed local fixture does not validate live EKS, ECR, ALB, or Jenkins delivery.
