# Jenkins Pipeline for Hextris

This repository contains a declarative Jenkins pipeline (`Jenkinsfile`) that builds and publishes container images for the Hextris application to the DigitalOcean Container Registry. The pipeline is designed for a Jenkins controller that can schedule builds onto a Kubernetes cluster, and it expects supporting tooling (Kaniko, Skopeo, Helm, Kubectl) to be provided via a multi-container pod.

---

## Pipeline Overview
- **Purpose:** Build Docker images for the `hextris-app` project and push them to `registry.digitalocean.com/ruhi-creg` tagged with both the short Git SHA and `latest`.
- **Idempotency:** If the image for a specific commit already exists, the pipeline skips the build to avoid unnecessary work.
- **Security:** Uses a Jenkins credential (`do-api-token`) to authenticate with the DigitalOcean registry through Skopeo and Kaniko.

---

## Infrastructure & Credentials Prerequisites
- **Jenkins requirements:**
  - Jenkins configured with the Kubernetes plugin (or an equivalent cloud agent provider).
  - A `default` Kubernetes pod template that the pipeline can inherit base settings from.
  - A shared library or global configuration that enables the `pipeline` step to launch custom YAML-defined pods.
- **Kubernetes cluster:** The Jenkins controller must be able to create pods in the target namespace. Cluster needs outbound internet access to pull Kaniko, Skopeo, Helm, and Kubectl images.
- **Credentials:**
  - `do-api-token`: Jenkins secret text containing a DigitalOcean access token with permission to interact with the container registry.
- **Source control access:** Jenkins must be able to clone `https://github.com/AlRuh-ux/devops-assignment.git`.
- **Image repository:** The DigitalOcean registry `registry.digitalocean.com/ruhi-creg` must exist, and the authenticated user must have push/pull permissions.

---

## Kubernetes Agent Pod Layout

When the pipeline starts, it spins up a Kubernetes pod labeled `hextris-agent` with four sidecar containers:

| Container | Image | Purpose |
| --- | --- | --- |
| `kaniko` | `gcr.io/kaniko-project/executor:debug` | Builds and pushes images without requiring Docker-in-Docker. Mounts `/kaniko/.docker` for registry auth. |
| `skopeo` | `quay.io/skopeo/stable:latest` | Inspects remote registries, used to check if an image already exists. |
| `helm` | `alpine/helm:3.14.0` | Available for future deployment stages (not used in the current Jenkinsfile). |
| `kubectl` | `bitnami/kubectl:latest` | Available for future rollout or verification steps. |

A temporary `emptyDir` volume named `docker-config` is mounted into the Kaniko container so the pipeline can inject a registry authentication file at runtime.

All containers start with a `sleep` command to stay alive while Jenkins attaches to them as needed.

---

## Environment Variables
- `REGISTRY`: DigitalOcean registry base URL (`registry.digitalocean.com/ruhi-creg`).
- `IMAGE_NAME`: Logical name for the Hextris image (`hextris-app`).
- `GIT_COMMIT_SHORT`: Populated during the checkout stage with the short 7-character Git SHA of the build.
- `SKIP_BUILD`: Flag used to determine whether the `Build and Push` stage should run. Defaults to unset, then explicitly set to `"true"` or `"false"` during runtime.

---

## Stage-by-Stage Breakdown

### Checkout
- **Action:** Uses the Jenkins `checkout` step to clone the `main` branch from GitHub.
- **Why it matters:** Ensures the workspace contains the latest source code and Docker context.
- A scripted block captures the short commit SHA (`git rev-parse --short=7 HEAD`) and echoes the value for traceability.

### Check if Image Exists
- **Container:** Runs inside the `skopeo` container to interact with container registries.
- **Credentials:** Wraps the Skopeo command with `withCredentials` to inject `DO_API_TOKEN` into the environment.
- **Logic:** Executes `skopeo inspect` against `docker://registry.digitalocean.com/ruhi-creg/hextris-app:<short-sha>`.
  - If the inspect command succeeds, the pipeline sets `SKIP_BUILD=true` and prints a skip message.
  - If the inspect command fails (image missing), it sets `SKIP_BUILD=false`, indicating that the build must proceed.
- **Benefit:** Prevents redundant builds and pushes for previously processed commits.

### Build and Push Image
- **Condition:** Controlled by `when { environment name: 'SKIP_BUILD', value: 'false' }`. Runs only if the image is absent.
- **Container:** Uses the `kaniko` container; Kaniko builds OCI images without requiring Docker daemon privileges.
- **Working directory:** Changes into `hextris-app` so Kaniko has direct access to the application Dockerfile and context.
- **Authentication:** Dynamically generates `/kaniko/.docker/config.json` with base64 credentials derived from `do-api-token`.
- **Kaniko execution:**
  - Builds the image using the local `Dockerfile`.
  - Pushes to two tags: `<short-sha>` and `latest`.
  - Enables Kaniko caching (`--cache=true`) and cleans up layers after completion (`--cleanup`).
- **Outcome:** A fresh application image is published to the registry, ready for deployment.

---

## Post Actions
- `post { success { ... } }`: Reports whether the build was skipped or a new image was published, including the final tag.
- `post { failure { ... } }`: Emits a clear failure message, prompting users to inspect Jenkins logs for the root cause.

---

## Real-Time Usage Scenarios

### Scenario 1: First-Time Build of a New Commit
1. Developer merges feature code into `main`.
2. Jenkins triggers the pipeline; the pod spins up with Kaniko and Skopeo.
3. Checkout stage pulls the latest `hextris-app` code; `GIT_COMMIT_SHORT` might be `a1b2c3d`.
4. Skopeo checks for `registry.digitalocean.com/ruhi-creg/hextris-app:a1b2c3d`. The tag is missing, so `SKIP_BUILD=false`.
5. Build stage runs: Kaniko builds the Docker image from `hextris-app/Dockerfile`, pushes both `:a1b2c3d` and `:latest`.
6. Success block announces the new tags. Deployment automation (e.g., Argo CD or Helm) can now pull the fresh image.

### Scenario 2: Re-running for an Existing Commit
1. A maintainer manually replays the pipeline for commit `a1b2c3d`.
2. Checkout repeats and the same short SHA is derived.
3. Skopeo detects that `:a1b2c3d` already exists in the registry, so `SKIP_BUILD=true`.
4. Build stage is skipped, saving cluster resources and time.
5. Success block logs that the image already exists - no further action required.

---

## Operational Notes & Customization Tips
- **Changing the Registry or Image Name:** Update the `REGISTRY` and `IMAGE_NAME` environment variables at the top of the Jenkinsfile. Keep tags consistent with downstream deployment tooling.
- **Altering Branch or Repository:** Modify the `url` or `branches` entries in the `checkout` stage if the mainline source lives elsewhere.
- **Enabling Deployments:** The pod already includes Helm and Kubectl containers. You can append additional stages after the build to run `helm upgrade` or `kubectl rollout` commands.
- **Tuning Kaniko:** Adjust caching flags, add `--build-arg` entries, or change the Dockerfile path if needed.
- **Credential Rotation:** Update the Jenkins `do-api-token` secret whenever the DigitalOcean token changes. The pipeline reads it dynamically, so no code changes are required.
- **Resource Management:** The pod inherits from Jenkins' `default` template. Adjust resource requests/limits, node selectors, or tolerations in the YAML block to tailor performance.

---

## Troubleshooting Checklist
- **Checkout failures:** Verify GitHub access tokens (if required) and ensure the repository URL is correct.
- **Skopeo errors:** Confirm the `do-api-token` credential exists and has registry read permissions. Check that the registry and repository names match DigitalOcean's configuration.
- **Kaniko authentication issues:** Review the generated `/kaniko/.docker/config.json` (printed in debug logs if you add temporary echoes) and ensure the token is valid.
- **Push failures:** Make sure the registry project (`ruhi-creg`) exists and the token has write access. Retries may help if DigitalOcean experiences transient issues.
- **Tag mismatch:** If downstream systems expect semantic versioning, adjust the tagging logic to append or replace `latest` with your desired convention.
- **Pod startup problems:** Ensure the Kubernetes cluster allows pulling the Kaniko, Skopeo, Helm, and Kubectl images, and that Jenkins has permission to schedule pods.
