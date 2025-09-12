
# Review-Filmes — CI/CD Multienv com Jenkins, Kustomize e EKS

> Aplicação **.NET 8** com **PostgreSQL**, pipeline CI/CD multienv em **Jenkins**, integrando **Amazon ECR**, **Trivy**, **SonarQube**, **ArgoCD** e **Kustomize** para deploy no **Amazon EKS**.

---

## 🚦 Visão Geral

- **Branches**:
  - `develop` → ambiente **Homologação (hml)**
  - `main` → ambiente **Produção (prd)**

- A pipeline adapta variáveis, credenciais e imagens automaticamente de acordo com a branch.

---

## 🌳 Estrutura do Repositório

```text
.
├─ Jenkinsfile              # Pipeline multienv
├─ k8s/deploy/
│  ├─ base/                 # Manifests base
│  ├─ hml/                  # Overlay homologação
│  └─ prd/                  # Overlay produção
└─ src/                     # Aplicação .NET + testes
```

---

## ⚙️ Etapas da Pipeline

1. **Validar branch → stage**  
   - main → prd  
   - develop → hml  

2. **Checkout App** → baixa código da branch correspondente.  
3. **Compute SHA** → calcula hash do commit.  
4. **Build Docker Image** → build/push da imagem para ECR.  
5. **Trivy Scan** → scan de vulnerabilidades, relatório no S3.  
6. **Testes Unitários** → executa testes .NET, envia resultados para S3.  
7. **SonarQube Analysis** → análise estática, usando credencial `hml-sonar-token` ou `prd-sonar-token`.  
8. **Push Garantido** → garante a imagem no ECR.  
9. **Deploy via ArgoCD**  
   - Atualiza `kustomization.yaml` no overlay correto.  
   - Commit/push na branch do GitOps.  
   - Sync da aplicação no ArgoCD.  

---

## 🔐 Credenciais no Jenkins

- `aws` → credenciais AWS  
- `git` → credenciais GitHub  
- `hml-sonar-token` → token Sonar hml  
- `prd-sonar-token` → token Sonar prd  
- `argocd-token` → token ArgoCD  

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

## 📈 Relatórios & Artefatos

- **Trivy** → S3 `/trivy/`  
- **Testes Unitários** → S3 `/testes/`  

---

## 👥 Contribuição

1. `git checkout -b feature/minha-feature`  
2. Commits semânticos  
3. PR para `develop` (hml) ou `main` (prd)  

---

## 📄 Licença

MIT (ou outra da sua preferência)
