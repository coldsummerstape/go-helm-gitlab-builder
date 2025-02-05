---

## Logic Overview

The logic consists of the following stages:

1. **Preparation (.pre)**  
2. **Building binary files (build)**  
3. **Building Docker images (docker-build)**  
4. **Publishing the Helm chart (helm-push)**  

---

## 1. .pre Stage
This stage is required for automatically obtaining a semantic version (e.g., `0.0.1`).

> **Important**: `.pre` runs **always**, regardless of other pipeline conditions.

---

## 2. Build Stage (Compiling Go Binaries)
During the `build-go` stage, we compile Go applications for two different architectures:  
- **amd64** (standard 64-bit systems on Intel/AMD)  
- **arm64** (ARM processors).  

1. A Docker container is launched based on the Go image (the `GO_IMAGE` variable is specified).
2. Required dependencies (including `git`, `ssh`) are installed, and SSH is configured for repository access.
3. Go dependencies are updated (`go mod tidy`, `go mod vendor`).
4. Compilation is performed using `go build`, and the results are stored in `./.out/$GO_TARGET_OS-$GO_TARGET_ARCH/`.
5. The compiled binaries, along with `migrations` and `assets` folders (if present), are stored as artifacts for use in the next stage.

These artifacts are stored in GitLab for **one week**, allowing them to be downloaded or used in subsequent pipeline steps.

---

## 3. Docker Build Stage (Building Docker Images)
At this stage, each Go binary is converted into a Docker image. Under the hood, we use **Kaniko**, which allows building images without needing a running Docker daemon.

### 3.1 Preparing the Working Directory
From the previous stage's artifacts, we copy all necessary files (binaries, `migrations`, `assets`) into the working directory `$WORKSPACE_DIR`.

### 3.2 Generating a Dockerfile
For **each** compiled binary file, an individual Dockerfile is automatically created. The logic is simple:

1. **Base image** — `golang:1.23-alpine3.20` (lightweight Alpine Linux with required tools like Go, bash, git, etc.).
2. The Dockerfile includes necessary package installations (e.g., `mysql-client`, `grpcurl`).
3. The **specific** binary file for which the Dockerfile is generated is copied into the container.
4. If the binary is named `migrator`, it is responsible for database migrations, so we add:
   ```dockerfile
   COPY ./migrations /app/migrations
   ```
5. If the working directory contains an `assets` folder, we copy it as well:
   ```dockerfile
   COPY ./assets /app/assets
   ```
6. The final command to execute when the container starts:
   ```dockerfile
   CMD ["/app/<binary_name>"]
   ```
Each Dockerfile is saved alongside its corresponding binary file as `Dockerfile.<binary_name>`.

> **Example** (for a `migrator` binary with an `assets` folder):
> ```dockerfile
> FROM golang:1.23-alpine3.20
> RUN apk update && apk add --no-cache \
>     git bash curl vim mysql-client iproute2 && \
>     go install github.com/grpc-ecosystem/grpc-health-probe@latest && \
>     echo "https://dl-cdn.alpinelinux.org/alpine/edge/testing" >> /etc/apk/repositories && \
>     apk add grpcurl --update-cache
> 
> ENV PATH=/app:/usr/local/bin:/busybox
> COPY ./migrator /app/migrator
> COPY ./migrations /app/migrations
> COPY ./assets /app/assets
> CMD ["/app/migrator"]
> ```

### 3.3 Building the Docker Image with Kaniko
Once the Dockerfile is ready, Kaniko builds the final image and pushes it to GitLab Container Registry.

- The image name is typically formatted as:
  ```
  ${CI_REGISTRY_IMAGE}/${binary_name}:${TAG}
  ```
- `TAG` can either be the commit tag (`CI_COMMIT_TAG`) or a branch name-based tag (`CI_COMMIT_REF_SLUG`) if no tag exists.

---

## 4. helm-push Stage (Publishing the Helm Chart)
The final stage is updating and publishing the Helm chart.

1. **Chart Versioning**  
   - If a commit tag (`CI_COMMIT_TAG`) is present, it is used as the version.  
   - If no tag exists, the branch name (`CI_COMMIT_REF_NAME`) is used.

2. **Updating `Chart.yaml` and `values.yaml`**  
   - The new version is written to `Chart.yaml`.  
   - `values.yaml` is updated with the correct Docker image tags for both architectures (amd64 and arm64).

3. **Packaging and Uploading the Helm Chart**  
   - Using `helm package`, we create an archive with the chart.  
   - The chart is then uploaded to the GitLab Helm registry (or another Helm registry) via GitLab's HTTP API.

> **Note**: The `helm-push` step **only runs** if a commit tag is present (`rules: if: $CI_COMMIT_TAG`). If working on a branch without a tag, this stage is skipped.

---

## How It All Works Together (Simple Explanation)
1. **Building the binary**: We compile the Go source code into an executable file for the required architectures.
2. **Creating the Docker image**:  
   - A Dockerfile is generated for each binary, specifying:
     - The base image.
     - Required package installations.
     - Which folders (migrations, assets) to include.
     - The command to execute in the container.  
   - The Docker image is built and pushed to the registry.
3. **Publishing the Helm chart**:  
   - The Helm chart files are updated to match the new versions of the images.
   - The updated chart is uploaded to the Helm registry.

As a result, once everything is ready, any Kubernetes cluster configured to use this Helm registry can pull the chart and deploy the application as pods with the correct images (amd64 or arm64).

This provides a **fully automated pipeline**: from code changes in the repository to a deployable Helm chart with up-to-date Docker images.