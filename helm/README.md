# Hextris Helm Chart

This chart packages the Hextris web application for deployment onto a Kubernetes cluster. The manifests follow Helm best practices and are tuned for DigitalOcean Kubernetes (DOKS), including optional HTTPS termination with cert-manager.

---

## Repository Layout

```
hextris/
├── .helmignore             # Ignore rules when running `helm package` or `helm lint`
├── Chart.yaml              # Chart metadata: name, description, versioning
├── values.yaml             # Default configuration that templates read from
├── charts/                 # Dependency charts (empty unless you add subcharts)
└── templates/              # Rendered into Kubernetes manifests
    ├── _helpers.tpl        # Reusable template partials (names, labels, service accounts)
    ├── deployment.yaml     # Creates the app Deployment and Pods
    ├── ingress.yaml        # Optional Ingress for public routing and TLS
    └── service.yaml        # Exposes the Deployment internally via ClusterIP
```

### File-by-file details

- `Chart.yaml`: Declares Helm API version, chart name (`hextris`), chart type (`application`), semantic version (`0.1.0`), and the upstream application version (`1.16.0`). Helm reads this first to understand compatibility and version constraints.
- `.helmignore`: Patterns excluded when packaging or linting the chart, ensuring IDE metadata and VCS folders are not bundled.
- `values.yaml`: Houses the default input values consumed by templates. It defines:
  - `replicaCount`: Pod replicas for high availability (defaults to 2).
  - `image`: DigitalOcean Container Registry image location and pull policy.
  - `service`: ClusterIP service exposing the app on port 80.
  - `resources`: CPU and memory requests/limits sized for a small workload.
  - `ingress`: Fully configured Ingress with TLS and cert-manager annotations targeting `roohi.mecarvi.com`.
- `charts/`: Reserved for dependency charts (for example, adding Redis as a subchart later). Empty by default.
- `templates/_helpers.tpl`: Defines template helpers:
  - `hextris.name` / `hextris.fullname`: Generate names constrained to 63 characters.
  - `hextris.chart`: Produces the chart label.
  - `hextris.labels` / `hextris.selectorLabels`: Standard label sets for Deployments, Services, and Ingress.
  - `hextris.serviceAccountName`: Future-proofed helper should you introduce service accounts in `values.yaml`.
- `templates/deployment.yaml`: Renders a Deployment with the labels from `_helpers.tpl`. It injects the replica count, container image, pull policy, and resource limits from `values.yaml`. The container exposes port 80.
- `templates/service.yaml`: Builds a ClusterIP Service named after the release. It selects Pods via the generated labels and forwards traffic to container port 80.
- `templates/ingress.yaml`: Conditionally renders when `ingress.enabled` is true. Handles:
  - Ingress class assignment (e.g., DigitalOcean's managed Nginx).
  - TLS blocks derived from `values.yaml.ingress.tls`.
  - Multiple host/path rules, each pointing at the Service and port 80.

---

## Configuring the Chart

Override `values.yaml` to match your environment without editing the defaults in-place. Either edit a copy or supply overrides with `--set`/`--values`.

Common adjustments:

- `image.repository` / `image.tag`: Point to your container registry (DigitalOcean Container Registry or another provider).
- `replicaCount`: Scale horizontally once the app is stable.
- `service.type`: Switch to `LoadBalancer` if you prefer to expose the Service via a cloud load balancer instead of Ingress.
- `ingress.hosts[0].host`: Replace the sample FQDN (`roohi.mecarvi.com`) with your domain.
- `ingress.tls`: Ensure the secret name matches the one cert-manager should populate. Update hostnames to match the `hosts` list.
- `resources`: Right-size Pod requests/limits to avoid evictions or throttling.

Example override file:

```yaml
# overrides.yaml
image:
  repository: registry.digitalocean.com/my-registry/hextris-app
  tag: "2024.04.15"

ingress:
  hosts:
    - host: game.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: hextris-prod-tls
      hosts:
        - game.example.com
```

---

## Prerequisites for DigitalOcean Kubernetes (DOKS)

1. **DigitalOcean account and API token** with read/write access.
2. **DOKS cluster** created and reachable.
3. **Helm 3.x** installed locally.
4. **kubectl** configured to talk to your DOKS cluster.
5. **doctl CLI** (optional but recommended) for managing cluster auth and registry.
6. **DigitalOcean Container Registry** containing the Hextris image referenced in `values.yaml`. Authenticate your cluster/namespace with `doctl registry kubernetes-manifest` or create a pull secret manually.
7. **Ingress controller** (DigitalOcean installs Nginx by default) and **cert-manager** with a `ClusterIssuer` named `letsencrypt-prod` if you keep the default TLS annotations. Adjust the issuer name if your setup differs.

---

## Deployment Workflow on DOKS

1. **Authenticate with DigitalOcean**
   ```powershell
   doctl auth init --context hextris
   ```
   Set `DIGITALOCEAN_ACCESS_TOKEN` beforehand or paste it when prompted.

2. **Fetch kubeconfig for the cluster**
   ```powershell
   doctl kubernetes cluster kubeconfig save <cluster-name-or-id> --context hextris
   kubectl config use-context <doks-context-name>
   kubectl get nodes
   ```
   Confirm nodes list successfully.

3. **Allow the cluster to pull from the container registry**
   - If using DigitalOcean Container Registry:
     ```powershell
     doctl registry kubernetes-manifest | kubectl apply -f -
     kubectl patch serviceaccount default -p "{\"imagePullSecrets\": [{\"name\": \"registry-ruhi-creg\"}]}" (CMD)
     ```
   - For other registries, create a Docker registry secret and reference it via `spec.imagePullSecrets` (add to the chart if required).

4. **Install or upgrade the release**
   ```powershell
   helm upgrade --install hextris ./hextris --namespace hextris --create-namespace`
   ```
   Replace `hextris` with your desired release name. Namespaces are created automatically using `--create-namespace`.

5. **Verify the rollout**
   ```powershell
   kubectl get deploy,svc,ingress -n hextris
   kubectl rollout status deploy/hextris -n hextris
   ```
   Wait for pods to reach `Ready` status and confirm the Ingress has an assigned address.

6. **Check DNS and TLS**
   - Point your domain’s DNS `A` record (or CNAME) to the Ingress load balancer IP.
   - Validate certificate issuance with `kubectl describe certificate -n hextris`.
   - Test HTTPS access in a browser once DNS propagates.

---

## Maintenance Tips

- **Upgrades**: Bump the image tag or configuration in your override file, then rerun the `helm upgrade --install` command.
- **Rollbacks**: Use `helm rollback hextris <revision>` if a release fails.
- **Uninstall**: `helm uninstall hextris --namespace hextris` removes all chart-managed resources.
- **Observability**: Integrate metrics/logging (e.g., enable DigitalOcean's observability stack) and consider enabling readiness/liveness probes in `deployment.yaml` for production hardening.

---

## Troubleshooting

- **Image pull errors**: Ensure the registry secret exists in the namespace and the image tag is correct.
- **Ingress not reachable**: Confirm Nginx Ingress controller is running, DNS resolves to the load balancer, and firewall rules allow traffic.
- **TLS issues**: Check cert-manager logs and ensure the `ClusterIssuer` referenced in annotations is installed and healthy.
- **Resource throttling**: Adjust resource requests/limits in `values.yaml` to match workload demands.

With these instructions and explanations, you should be able to understand each component of the chart and confidently deploy Hextris onto a DigitalOcean Kubernetes cluster.

