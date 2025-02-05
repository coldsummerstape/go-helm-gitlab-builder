---

# **Go Helm GitLab Builder** 🏗️🐳🚀  

**Automated CI/CD pipeline** for **Go applications** with **Docker & Helm** using **GitLab CI/CD**  

---

## 🌟 **Pipeline Overview**  

🚀 **This pipeline automates:**  
✅ Building Go binaries for **multiple architectures** (amd64 & arm64)  
✅ Creating **Docker images** using **Kaniko**  
✅ Publishing **Helm charts** for Kubernetes deployments  

🛠 **Technologies used:**  
- 🔹 **Go** (Golang)  
- 🔹 **GitLab CI/CD**  
- 🔹 **Kaniko** (for Docker builds)  
- 🔹 **Helm** (for Kubernetes deployments)  

---

## 🔄 **Pipeline Stages**  

| 🔢 Stage          | 🔧 Description |
|------------------|------------------------------|
| **1️⃣ .pre**    | Generates a **semantic version** (e.g., `0.0.1`) ✅ Always runs |
| **2️⃣ Build**   | Compiles **Go binaries** for `amd64` & `arm64` |
| **3️⃣ Docker**  | Builds **Docker images** with **Kaniko** |
| **4️⃣ Helm**    | Updates & publishes **Helm charts** |

---

## 🏗️ **1. .pre Stage: Versioning**  

📌 **Goal:** Automatically obtain a **semantic version** for the pipeline (e.g., `0.0.1`).  
📌 **Note:** This stage **always runs**, regardless of other pipeline conditions.  

```yaml
.pre_get_unique_semversion:
  stage: .pre
  script:
    - echo "Retrieving a unique semantic version"
  rules:
    - when: always
```

---

## 🏗️ **2. Build Stage: Compiling Go Binaries**  

📌 **Goal:** Compile Go applications for **two different architectures**:  
- ✅ **amd64** (64-bit Intel/AMD)  
- ✅ **arm64** (ARM-based processors)  

📌 **Key Steps:**  
✔️ Use a **Go Docker container**  
✔️ Install dependencies (**Git, SSH**)  
✔️ Run **Go mod tidy** & **Go mod vendor**  
✔️ **Compile** binaries and store them in `./.out/$GO_TARGET_OS-$GO_TARGET_ARCH/`  

📌 **Artifacts:**  
📁 Compiled **binaries**  
📁 **Migrations** & **assets** (if available)  

⏳ **Retention:** **Stored for 1 week** in GitLab  

---

## 🏗️ **3. Docker Build Stage: Containerizing Binaries**  

📌 **Goal:** Convert Go binaries into **Docker images** using **Kaniko** 🚀  

📌 **Steps:**  
✔️ Copy artifacts (`binaries`, `migrations`, `assets`) to **`$WORKSPACE_DIR`**  
✔️ **Auto-generate Dockerfiles** for each binary  
✔️ Use **Kaniko** to **build & push** images to **GitLab Container Registry**  

📌 **Example Dockerfile:**  
```dockerfile
FROM golang:1.23-alpine3.20
RUN apk update && apk add --no-cache \
    git bash curl vim mysql-client iproute2 && \
    go install github.com/grpc-ecosystem/grpc-health-probe@latest && \
    echo "https://dl-cdn.alpinelinux.org/alpine/edge/testing" >> /etc/apk/repositories && \
    apk add grpcurl --update-cache

ENV PATH=/app:/usr/local/bin:/busybox
COPY ./migrator /app/migrator
COPY ./migrations /app/migrations
COPY ./assets /app/assets
CMD ["/app/migrator"]
```

📌 **Image Naming Convention:**  
```
${CI_REGISTRY_IMAGE}/${binary_name}:${TAG}
```
✔️ `TAG = CI_COMMIT_TAG` (if available)  
✔️ Else, `TAG = CI_COMMIT_REF_SLUG`  

---

## 🏗️ **4. Helm-push Stage: Publishing Helm Charts**  

📌 **Goal:** Update & publish **Helm charts** for Kubernetes deployments.  

📌 **Steps:**  
✔️ **Set chart version** based on GitLab CI/CD variables  
✔️ **Update `Chart.yaml` & `values.yaml`**  
✔️ **Package & upload** the Helm chart to GitLab  

📌 **Runs only when a Git tag is present:**  
```yaml
rules:
  - if: '$CI_COMMIT_TAG'
```

📌 **Retention:**  
📁 **Helm Charts** stored in **GitLab Helm Registry**  

---

## 📌 **How It All Works Together** 🏗️  

1️⃣ **Build Binaries** 🏗️  
✅ Compile Go binaries for `amd64` & `arm64`  

2️⃣ **Create Docker Images** 🐳  
✅ Auto-generate `Dockerfile` for each binary  
✅ Build & push images to **GitLab Container Registry**  

3️⃣ **Publish Helm Chart** 📦  
✅ Update `values.yaml` with the new Docker image tags  
✅ Upload chart to **GitLab Helm Registry**  

4️⃣ **Deploy on Kubernetes** 🚀  
✅ Any **Kubernetes cluster** can pull & deploy the Helm chart  

---

## 🎯 **Why Use This Pipeline?**  

✅ **Fully Automated Deployment** 🔄  
✅ **Supports Multi-Arch (amd64 & arm64)** 🏗️  
✅ **Lightweight & Fast (Alpine Linux + Kaniko)** 🐳  
✅ **Integrates Seamlessly with GitLab CI/CD** 🎯  

🚀 **Perfect for deploying Go apps in Kubernetes with Helm!**  

---

### **📜 License**  
This project is **open-source** and licensed under the **MIT License**.  

---

## 🤝 **Contributing**  

Contributions are welcome! Feel free to **open issues** or **submit PRs**.  

📧 **Contact:** `coldsummerstape` on **GitHub**  

🚀 **Happy Coding!** 🚀  

---

### **🔥 Bonus: Quick Setup**
Want to get started? Clone this repository and modify `.gitlab-ci.yml` for your **Go project**! 🎯  

```sh
git clone git@github.com:coldsummerstape/go-helm-gitlab-builder.git
cd go-helm-gitlab-builder
git checkout main
```

---