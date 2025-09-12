# Review-Filmes — Multienv CI/CD with Jenkins, Kustomize and EKS

> **.NET 8** application with **PostgreSQL**, multienv CI/CD pipeline in **Jenkins**, integrating **Amazon ECR**, **Trivy**, **SonarQube**, **ArgoCD**, and **Kustomize** for deployment on **Amazon EKS**.

---

## 🚦 Overview

- **Branches**:
  - `develop` → **Staging (hml)** environment
  - `main` → **Production (prd)** environment

- Pipeline automatically switches variables, credentials, and images according to branch.

---

## 🌳 Repository Structure

```text
.
├─ Jenkinsfile              # Multienv pipeline
├─ k8s/deploy/
│  ├─ base/                 # Base manifests
│  ├─ hml/                  # Staging overlay
│  └─ prd/                  # Production overlay
└─ src/                     # .NET app + tests
```

---

## ⚙️ Pipeline Stages

1. **Validate branch → stage**  
   - main → prd  
   - develop → hml  

2. **Checkout App** → fetch code from branch.  
3. **Compute SHA** → compute commit hash.  
4. **Build Docker Image** → build/push to ECR.  
5. **Trivy Scan** → vulnerability scan, report to S3.  
6. **Unit Tests** → run .NET tests, upload results to S3.  
7. **SonarQube Analysis** → static analysis using `hml-sonar-token` or `prd-sonar-token`.  
8. **Guaranteed Push** → ensures image in ECR.  
9. **Deploy via ArgoCD**  
   - Updates `kustomization.yaml` in correct overlay.  
   - Commit/push to GitOps branch.  
   - Sync application in ArgoCD.  

---

## 🔐 Jenkins Credentials

- `aws` → AWS credentials  
- `git` → GitHub credentials  
- `hml-sonar-token` → Sonar staging token  
- `prd-sonar-token` → Sonar production token  
- `argocd-token` → ArgoCD token  

---

## ☸️ Kubernetes & ArgoCD

- Namespaces:
  - `hml-reviewfilmes`
  - `prd-reviewfilmes`

- Ingress:
  - HML → `homolog.app.wsnobrega.life`
  - PRD → `prod.app.wsnobrega.life`

- ECR Repos:
  - `216989136189.dkr.ecr.us-east-1.amazonaws.com/hml-review-filmes`
  - `216989136189.dkr.ecr.us-east-1.amazonaws.com/prd-review-filmes`

- ArgoCD Apps:
  - `hml-review-filmes`
  - `prd-review-filmes`

---

## 📈 Reports & Artifacts

- **Trivy** → S3 `/trivy/`  
- **Unit Tests** → S3 `/testes/`  

---

## 👥 Contribution

1. `git checkout -b feature/my-feature`  
2. Semantic commits  
3. PR to `develop` (hml) or `main` (prd)  

---

## 📄 License

MIT (or your choice)
