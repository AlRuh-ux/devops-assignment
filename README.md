# Hextris Deployment Playbook

This README documents the exact workflow used to provision infrastructure on DigitalOcean, push container images, install the required Kubernetes add-ons, and deploy the Hextris web application.

---

## Repository Layout
- `terraform-code/` &mdash; Infrastructure-as-Code that creates the managed Kubernetes cluster (`main.tf`, `variables.tf`, `terraform.tfvars`).
- `hextris-app/` &mdash; Static Hextris web assets and the Docker build recipe (`hextris-app/Dockerfile`).
- `helm/hextris/` &mdash; Helm chart that deploys the application, complete with ingress and TLS wiring (`helm/hextris/values.yaml`, `helm/hextris/templates/ingress.yaml`, etc.).
- `cluster-issuer.yaml` &mdash; Cluster-wide ACME issuer consumed by cert-manager (`cluster-issuer.yaml`).
- `Jenkinsfile` &mdash; CI pipeline definition that containerizes and publishes the app (`Jenkinsfile`). A focused explanation sits in `README-Jenkinsfile.md`.

---

## Prerequisites
Before running the workflow end to end, you need:

- A DigitalOcean account with API access and the `doctl` CLI authenticated (`doctl auth init`).
- Terraform CLI v1.3+ available on your workstation.
- Kubernetes CLI (`kubectl`) and Helm 3 installed.
- Docker Engine (or another OCI-compatible builder) for local image builds and validation.
- Access to manage DNS for `roohi.mecarvi.com` (used by the ingress and certificate).
- Jenkins with Kubernetes agents.

### Terraform Installation (Windows example)
1. Download the current Terraform ZIP from https://developer.hashicorp.com/terraform/downloads.
2. Extract it to a folder such as `C:\HashiCorp\Terraform`.
3. Add that folder to the `Path` user environment variable.
4. Close and reopen PowerShell, then verify:
   ```powershell
   terraform -version
   ```

Repeat analogous steps on macOS/Linux (e.g., via Homebrew, apt, or direct binary download).

---

## Phase 1 &mdash; Provision the Kubernetes Cluster

### 1. Set the DigitalOcean Token
The Terraform configuration expects the variable `do_token` (`terraform-code/variables.tf`). Export it before running Terraform so secrets are not hard-coded in `terraform.tfvars`.

```powershell
$env:TF_VAR_do_token = "dop_v1_your_real_token"
```

> **Important:** The sample `terraform-code/terraform.tfvars` currently contains a concrete token. Replace it with a placeholder or delete the file before committing to avoid leaking credentials.

### 2. Initialize and Plan
Change into the Terraform directory and download the DigitalOcean provider defined in `terraform-code/main.tf`.

```powershell
cd terraform-code
terraform init
terraform plan
```

The plan should show the creation of one `digitalocean_kubernetes_cluster.ruhi_k8s` resource with a single node pool (`worker-pool`) sized `s-1vcpu-2gb`.

### 3. Apply the Configuration
Provision the cluster without interactive approval:

```powershell
terraform apply --auto-approve
```

Terraform records the cluster ID and metadata in `terraform-code/terraform.tfstate`. Keep this file safe; it controls future updates and `terraform destroy`.

---

## Phase 2 &mdash; Retrieve the Kubeconfig

Use `doctl` to pull credentials for the new cluster. Replace the UUID with the value printed by the output of `doctl kubernetes cluster list`.

```powershell
doctl kubernetes cluster kubeconfig save 7a0911a9-dc10-4c9d-8e51-ac9867be8bc0
```

The command merges the context into your default kubeconfig (typically `%USERPROFILE%\.kube\config`).

---

## Phase 3 &mdash; Validate Cluster Access

Confirm that kubectl sees the new context (`do-sgp1-ruhi-k8s-cluster` by default):

```powershell
kubectl config get-contexts
kubectl config use-context do-sgp1-ruhi-k8s-cluster
kubectl cluster-info
kubectl version
kubectl get nodes
```

A single worker node should appear in the `Ready` state.

---

## Phase 4 &mdash; Prepare the DigitalOcean Container Registry (DOCR)

### 1. Create the Registry

```powershell
doctl registry create ruhi-creg
```

If the registry already exists, DigitalOcean will return a conflict error. Treat the command as idempotent.

### 2. Authenticate Docker Against DOCR

```powershell
doctl registry login
```

This populates your Docker credential helper so `docker push` can authenticate.

### 3. Authorize the Kubernetes Cluster
Generate the pull secret YAML and apply it straight to the cluster:

```powershell
doctl registry kubernetes-manifest | kubectl apply -f -
```

This creates a secret named `registry-ruhi-creg` in each namespace where you apply it; by default it lands in `default`.

### 4. Patch the Default ServiceAccount
Ensure Pods automatically attach the image pull secret. Both Windows PowerShell and Bash-friendly commands are shown:

```powershell
kubectl patch serviceaccount default `
  -p "{\"imagePullSecrets\": [{\"name\": \"registry-ruhi-creg\"}]}"
```

```bash
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "registry-ruhi-creg"}]}'
```

---

## Phase 5 &mdash; Build and Publish the Hextris Image

### 1. Clone and Inspect the App
The Hextris source lives in `hextris-app/`. The Dockerfile (`hextris-app/Dockerfile`) uses the lightweight `nginx:alpine` base image and serves the static game assets from `/usr/share/nginx/html`.

### 2. Build the Image Locally

```powershell
cd hextris-app
docker build -t registry.digitalocean.com/ruhi-creg/hextris-app:latest .
```

Optional: Run the container to validate the game loads correctly:

```powershell
docker run --rm -p 8080:80 registry.digitalocean.com/ruhi-creg/hextris-app:latest
# Navigate to http://localhost:8080
```

### 3. Tag with the Git SHA (recommended)

```powershell
$commit = (git rev-parse --short=7 HEAD)
docker tag registry.digitalocean.com/ruhi-creg/hextris-app:latest `
  registry.digitalocean.com/ruhi-creg/hextris-app:$commit
```

### 4. Push Both Tags

```powershell
docker push registry.digitalocean.com/ruhi-creg/hextris-app:latest
docker push registry.digitalocean.com/ruhi-creg/hextris-app:$commit
```

> The Jenkins pipeline (`Jenkinsfile`) automates this build-and-push flow with Kaniko and Skopeo inside ephemeral Kubernetes agents. See **Phase 8** for details.

---

## Phase 6 &mdash; Install Kubernetes Add-ons

### 1. NGINX Ingress Controller

```powershell
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx
kubectl get svc -n default
```

Wait for the `ingress-nginx-controller` service to obtain an external load balancer IP. Point the DNS `A` record for `roohi.mecarvi.com` to that IP.

### 2. cert-manager

```powershell
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true
```

### 3. ClusterIssuer
Apply the ACME issuer defined in `cluster-issuer.yaml`:

```powershell
kubectl apply -f ..\cluster-issuer.yaml
kubectl get clusterissuer
```

The issuer must be in the `Ready` state before continuing.

---

## Phase 7 &mdash; Deploy Hextris with Helm

### 1. Review Chart Values
`helm/hextris/values.yaml` defines:
- `image.repository`: `registry.digitalocean.com/ruhi-creg/hextris-app`
- `image.tag`: `latest` (override with a specific commit tag for deterministic releases)
- `ingress.hosts[0].host`: `roohi.mecarvi.com`
- `ingress.tls[0].secretName`: `hextris-tls`

Adjust these values if you operate in a different registry, namespace, or domain.

### 2. Install or Upgrade

```powershell
cd helm
helm upgrade --install hextris ./hextris --namespace hextris --create-namespace
helm list -A
helm get manifest hextris -n hextris
```

### 3. Verify Workloads

```powershell
kubectl get pods -n hextris
kubectl get svc -n hextris
kubectl get ingress -n hextris
kubectl get certificate -n hextris
kubectl get secret hextris-tls -n hextris -o yaml
```

Expect two application pods (replica count in `values.yaml`), a ClusterIP service, an ingress resource with the configured host, and a certificate in `Ready` status backed by LetsEncrypt.

### 4. Functional Check

Visit https://roohi.mecarvi.com in a browser. You should reach the Hextris UI over HTTPS with a valid TLS certificate.

---

## Phase 8 &mdash; Continuous Delivery with Jenkins

The declarative pipeline in `Jenkinsfile` automates image builds and pushes:

- **Agent Model:** Each run spins up a temporary Kubernetes pod (`kaniko`, `skopeo`, `helm`, `kubectl` containers) and tears it down after completion.
- **Registry Check:** A Skopeo step inspects DOCR for the current Git commit tag. If the tag already exists, the build is skipped.
- **Image Build:** Kaniko builds `hextris-app` and pushes both the commit SHA tag and `latest`.
- **Credentials:** The pipeline consumes the `do-api-token` secret from Jenkins credentials to authenticate with DigitalOcean.

---

## Maintenance

- **Scaling:** Increase `replicaCount` or resource requests in `helm/hextris/values.yaml` and redeploy with Helm.
- **Certificate Renewal:** cert-manager automatically renews LetsEncrypt certificates. Ensure the ingress remains reachable and DNS stays aligned.
- **Upgrades:** Use `terraform plan` to pick a new Kubernetes version or resize node pools (`terraform-code/main.tf`). Run `terraform apply` to enact changes.
- **Cleanup:** To tear everything down:
  ```powershell
  helm uninstall hextris -n hextris
  helm uninstall ingress-nginx
  helm uninstall cert-manager -n cert-manager
  terraform -chdir=terraform-code destroy
  doctl registry delete ruhi-creg
  ```
  Remove DNS records that pointed to the ingress.

---

## Troubleshooting

- **Pods stuck in ImagePullBackOff:** Confirm the default service account has `registry-ruhi-creg` attached and that the image tag exists in DOCR.
- **Ingress has no external IP:** Check the DigitalOcean load balancer quota and describe the ingress controller service (`kubectl describe svc ingress-nginx-controller`).
- **Certificate stays in Pending:** Verify the ClusterIssuer status and that HTTP-01 challenges reach the ingress (inspect events on the ingress resource).
- **Terraform errors about credentials:** Make sure `TF_VAR_do_token` is set in the shell running Terraform. Do not rely on the sample `terraform.tfvars` value.
- **Jenkins pipeline cannot push images:** Ensure the Jenkins credential matches the token used locally and that Kaniko can write `/kaniko/.docker/config.json` as implemented in the `Build and Push Image` stage.

This playbook captures the full deployment lifecycle so future runs can recreate the environment from scratch with predictable results.
