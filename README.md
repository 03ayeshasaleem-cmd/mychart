# Deploying a Helm Chart with Argo CD on Kubernetes

## 1. Project Overview

This project demonstrates how to create a Helm chart, store it in GitHub, and deploy it to Kubernetes using Argo CD.

Argo CD follows the GitOps approach, where Git is the source of truth for the desired Kubernetes configuration.

### Project objectives

- Create a Helm chart using Helm.
- Validate the chart before deployment.
- Push the chart to GitHub.
- Install and configure Argo CD.
- Create an Argo CD Application.
- Deploy Kubernetes resources through Argo CD.
- Verify Deployments, Pods, Services, and endpoints.
- Access the deployed application through a web browser.

## 2. Technology Stack

| Technology | Purpose |
|---|---|
| Kubernetes | Container orchestration |
| Helm | Kubernetes package management |
| Git | Version control |
| GitHub | Store Helm chart and configuration |
| Argo CD | GitOps-based deployment |
| kubectl | Manage and inspect Kubernetes resources |
| WSL | Run Linux commands on Windows |

## 3. Project Architecture

```text
Local Helm Chart
       |
       v
     Git
       |
       v
    GitHub
       |
       v
    Argo CD
       |
       v
Kubernetes Cluster
       |
       v
Deployment
       |
       v
  Pods (3 replicas)
       |
       v
    Service
       |
       v
Browser: localhost:8081
```

## 4. Verify the Kubernetes Cluster

First, verify the active Kubernetes context and cluster nodes.

```bash
kubectl config current-context
kubectl get nodes
```

Make sure the intended cluster is active and its nodes are ready.

## 5. Create a Helm Chart

Create a new Helm chart named `mychart`.

```bash
helm create mychart
cd mychart
```

A typical Helm chart structure looks like this:

```text
mychart/
├── Chart.yaml
├── values.yaml
├── charts/
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    ├── hpa.yaml
    ├── ingress.yaml
    ├── NOTES.txt
    └── _helpers.tpl
```

### Validate the chart

```bash
helm lint .
helm template mychart-app .
```

- `helm lint .` checks the chart for common issues.
- `helm template mychart-app .` renders Kubernetes manifests locally without deploying them.

## 6. Push the Helm Chart to GitHub

Create a GitHub repository named `mychart`.

For this setup, the chart files are stored at the repository root, where `Chart.yaml` is located.

From inside the chart directory, run:

```bash
git init
git add .
git commit -m "Add Helm chart"
git branch -M main
```

Connect the local chart to GitHub:

```bash
git remote add origin https://github.com/03ayeshasaleem-cmd/mychart.git
git push -u origin main
```

If the Git remote already exists, check it instead of adding it again:

```bash
git remote -v
```

Verify that the GitHub repository contains:

- `Chart.yaml`
- `values.yaml`
- `templates/`
- `charts/` (if applicable)

**Important:** Do not store passwords or access tokens in the repository.

## 7. Install Argo CD

Create the Argo CD namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply --server-side --force-conflicts -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Verify the installation:

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

Wait for the required Argo CD components to become ready.

## 8. Access the Argo CD Dashboard

Run the following command in a terminal:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Keep this terminal open.

Open the following URL in your browser:

```text
https://localhost:8080
```

### Get the initial admin password

Open a second terminal and run:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
```

Log in using:

- **Username:** `admin`
- **Password:** The value returned by the command

The browser may display a certificate warning because the default installation uses a self-signed certificate.

## 9. Create the Argo CD Application

Create a file named `argocd-app.yaml` in your working directory.

Add the following configuration:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mychart-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/03ayeshasaleem-cmd/mychart.git
    targetRevision: main
    path: .
    helm:
      releaseName: mychart-app
  destination:
    server: https://kubernetes.default.svc
    namespace: mychart-demo
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

### Configuration explanation

- `metadata.name`: Name of the Argo CD Application.
- `metadata.namespace`: Namespace where the Argo CD Application resource exists.
- `repoURL`: GitHub repository containing the Helm chart.
- `targetRevision`: Git branch Argo CD should monitor.
- `path: .`: The chart is located at the repository root.
- `helm.releaseName`: Name used for Helm-rendered resources.
- `destination.namespace`: Namespace where the application resources are deployed.
- `CreateNamespace=true`: Allows Argo CD to create the destination namespace if it does not exist.

If the chart is inside a subdirectory, update `path` to that directory.

Apply the Application:

```bash
kubectl apply -f argocd-app.yaml
```

Verify it:

```bash
kubectl get applications -n argocd
kubectl describe application mychart-app -n argocd
```

## 10. Sync the Application

Initially, Argo CD may report the application as `OutOfSync`. This means the live cluster does not yet match the desired state from Git.

Request a sync:

```bash
kubectl patch application mychart-app -n argocd \
  --type merge \
  -p '{"operation":{"sync":{}}}'
```

Check the application status:

```bash
kubectl get application mychart-app -n argocd
```

Inspect the deployed resources:

```bash
kubectl get all -n mychart-demo
```

Check Deployments, Pods, and Services:

```bash
kubectl get deployments,pods,services -n mychart-demo
```

A successful deployment should eventually show the application as `Synced` and `Healthy`, provided its resources are healthy.

## 11. Verify the Deployment

Check the Service:

```bash
kubectl get svc -n mychart-demo
```

Check the Pods:

```bash
kubectl get pods -n mychart-demo
```

Check the Service endpoints:

```bash
kubectl get endpoints -n mychart-demo
```

The observed project state included:

- Service name: `mychart-app`
- Service type: `LoadBalancer`
- Service port: `8081`
- NodePort: `32058`
- Three application Pods
- All three Pods in `Running` state
- Three Service endpoints on port `8081`

These values reflect the observed cluster state and may change after future deployments.

## 12. Access the Application in a Browser

To access the application locally, run:

```bash
kubectl port-forward -n mychart-demo svc/mychart-app 8081:8081
```

Keep the terminal open while using the application.

Open this URL in your browser:

**http://localhost:8081**

The port-forward command maps local port `8081` to port `8081` on the selected Service's Pods.

In this project, the application became accessible through this command and URL.

> Note: If port-forwarding fails with `connection refused`, check whether the container process is listening on the Service's target port. A Pod can be `Running` even when no web server is listening.

## 13. Troubleshooting

### Check the Deployment

```bash
kubectl describe deployment mychart-app -n mychart-demo
```

### Check Pod details

```bash
kubectl describe pods -n mychart-demo
```

### Check application logs

```bash
kubectl logs deployment/mychart-app -n mychart-demo --tail=100
```

### Inspect the Service configuration

```bash
kubectl describe svc mychart-app -n mychart-demo
kubectl get svc mychart-app -n mychart-demo -o yaml
```

### Inspect the configured container image and ports

```bash
kubectl get deployment mychart-app -n mychart-demo \
  -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'

kubectl get deployment mychart-app -n mychart-demo \
  -o jsonpath='{.spec.template.spec.containers[*].ports}{"\n"}'
```

If the browser does not open, verify the application's listening port, the container logs, and the Service `targetPort`.

## 14. Update the Helm Chart Using GitOps

When you modify the chart, commit and push the changes to GitHub.

```bash
git add .
git commit -m "Update Helm chart"
git push
```

Argo CD compares the Git repository's desired configuration with the live Kubernetes resources.

If automated sync is not enabled, initiate a sync through the Argo CD dashboard or the command in Step 10.

Verify the updated resources:

```bash
kubectl get application mychart-app -n argocd
kubectl get pods -n mychart-demo
kubectl get svc -n mychart-demo
```

Git should remain the source of truth for the desired configuration. Prefer updating the chart in Git rather than making permanent manual changes directly to live resources.

## 15. Key Learnings

Through this project, I practised:

- Creating and validating Helm charts.
- Managing Kubernetes manifests with Helm templates.
- Using Git and GitHub for version control.
- Installing and accessing Argo CD.
- Creating an Argo CD Application.
- Deploying a Helm chart through GitOps.
- Verifying Kubernetes Deployments, Pods, Services, and endpoints.
- Accessing the application using `kubectl port-forward`.
- Troubleshooting application connectivity issues.

## 16. Conclusion

This project demonstrates a basic GitOps workflow in which a Helm chart is stored in GitHub and deployed to Kubernetes through Argo CD.

It connects Helm, GitHub, Argo CD, and Kubernetes into a repeatable deployment process and provides a foundation for more advanced CI/CD and GitOps projects.
