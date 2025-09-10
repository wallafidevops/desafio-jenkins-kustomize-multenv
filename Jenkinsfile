pipeline {
  agent any

  environment {
    // Comuns
    DOCKER_HOST           = 'tcp://localhost:2375'
    DOCKER_TLS_VERIFY     = ''
    AWS_DEFAULT_REGION    = 'us-east-1'
    AWS_REGION            = 'us-east-1'

    // Repositórios (monorepo: app + gitops)
    APP_REPO_URL          = 'https://github.com/wallafidevops/desafio-jenkins-kustomize-multenv.git'
    GITOPS_REPO_URL       = 'https://github.com/wallafidevops/desafio-jenkins-kustomize-multenv.git'

    // ECR
    AWS_ECR_URI           = '216989136189.dkr.ecr.us-east-1.amazonaws.com'

    // Sonar
    SONAR_HOST_URL        = 'https://sonarqube.app.wsnobrega.life'

    // Artefatos
    ARTIFACTS_BUCKET_PATH = 'wordpress-216989136189'

    // Git identity
    GIT_USER_NAME         = 'wallafidevops'
    GIT_USER_EMAIL        = 'wallafisantos55@gmail.com'

    // ArgoCD
    ARGOCD_SERVER         = 'argocd-server.argocd.svc.cluster.local'
    ARGOCD_TOKEN          = credentials('argocd-token')
  }

  stages {

    stage('Validar branch → stage') {
      steps {
        script {
          // main => prd | develop => hml | outras: erro
          if (env.BRANCH_NAME == 'main') {
            env.STAGE = 'prd'
            env.APP_BRANCH = 'main'
            env.GITOPS_BRANCH = 'main'
          } else if (env.BRANCH_NAME == 'develop') {
            env.STAGE = 'hml'
            env.APP_BRANCH = 'develop'
            env.GITOPS_BRANCH = 'develop'
          } else {
            error "Branch não permitida: ${env.BRANCH_NAME}. Use 'develop' (hml) ou 'main' (prd)."
          }

          // Variáveis com prefixo do ambiente
          env.IMAGE_NAME      = "${env.STAGE}-review-filmes"
          env.PROJECT_NAME    = "${env.STAGE}-desafio-jenkins-kustomize"
          env.ARGOCD_APP_NAME = "${env.STAGE}-review-filmes"
          env.SONAR_CRED_ID   = "${env.STAGE}-sonar-token" // hml-sonar-token / prd-sonar-token

          echo "STAGE=${env.STAGE} | APP_BRANCH=${env.APP_BRANCH} | GITOPS_BRANCH=${env.GITOPS_BRANCH}"
          echo "IMAGE_NAME=${env.IMAGE_NAME} | PROJECT_NAME=${env.PROJECT_NAME} | ARGOCD_APP_NAME=${env.ARGOCD_APP_NAME}"
          echo "SONAR_CRED_ID=${env.SONAR_CRED_ID}"
        }
      }
    }

    stage('Checkout app') {
      steps {
        git branch: "${env.APP_BRANCH}", credentialsId: 'git', url: env.APP_REPO_URL
      }
    }

    stage('Compute SHA') {
      steps {
        script {
          env.FULL_SHA = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
          echo "FULL_SHA=${env.FULL_SHA}"
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws']]) {
          sh '''
            set -eu
            aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$AWS_ECR_URI"

            echo "Build/Push -> $AWS_ECR_URI/$IMAGE_NAME:$FULL_SHA"
            cd src/
            docker build -t "$AWS_ECR_URI/$IMAGE_NAME:$FULL_SHA" --build-arg BUILD_ID="$FULL_SHA" .
            docker push "$AWS_ECR_URI/$IMAGE_NAME:$FULL_SHA"
          '''
        }
      }
    }

    stage('Trivy Scan') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws']]) {
          sh '''
            set -eu
            mkdir -p /tmp/trivy-cache ./reports
            echo "Trivy -> $AWS_ECR_URI/$IMAGE_NAME:$FULL_SHA"

            trivy image --scanners vuln --cache-dir /tmp/trivy-cache --no-progress \
              --format json -o ./reports/gl-container-scanning-report.json \
              "$AWS_ECR_URI/$IMAGE_NAME:$FULL_SHA"

            aws s3 cp ./reports/gl-container-scanning-report.json \
              "s3://$ARTIFACTS_BUCKET_PATH/trivy/gl-container-scanning-report-$(date +%F-%T).json"
          '''
        }
      }
    }

    stage('Testes Unitários') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws']]) {
          sh '''
            set -eu
            dotnet test src/Review-Filmes.Test.Unit/Review-Filmes.Test.Unit.csproj \
              --logger "trx;LogFileName=TestResults.trx" \
              --results-directory ./TestResults \
              --collect:"Code Coverage" || true

            aws s3 cp ./TestResults "s3://$ARTIFACTS_BUCKET_PATH/testes/" --recursive
          '''
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withCredentials([string(credentialsId: "${env.SONAR_CRED_ID}", variable: 'SONAR_TOKEN')]) {
          sh '''
            set -eu
            mkdir -p .tools
            dotnet tool install dotnet-sonarscanner --tool-path .tools || true
            export PATH="$PATH:$(pwd)/.tools"

            dotnet sonarscanner begin /k:"$PROJECT_NAME" /d:sonar.host.url="$SONAR_HOST_URL" /d:sonar.token="$SONAR_TOKEN"
            dotnet build src/Review-Filmes.sln
            dotnet sonarscanner end /d:sonar.token="$SONAR_TOKEN"
          '''
        }
      }
    }

    stage('Push Garantido') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws']]) {
          sh '''
            set -eu
            aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$AWS_ECR_URI"
            docker pull "$AWS_ECR_URI/$IMAGE_NAME:$FULL_SHA" || echo "Imagem ainda não existe, prosseguindo…"
            docker push "$AWS_ECR_URI/$IMAGE_NAME:$FULL_SHA"
          '''
        }
      }
    }

    stage('Deploy via ArgoCD (overlay por stage)') {
      steps {
        withCredentials([ usernamePassword(credentialsId: 'git', usernameVariable: 'GITHUB_USERNAME', passwordVariable: 'GITHUB_PASSWORD') ]) {
          sh '''
            set -eu

            # Git identity
            git config --global user.name "$GIT_USER_NAME"
            git config --global user.email "$GIT_USER_EMAIL"

            # Clone GitOps (monorepo)
            rm -rf gitops && mkdir -p gitops
            git clone "https://$GITHUB_USERNAME:$GITHUB_PASSWORD@${GITOPS_REPO_URL#https://}" gitops
            cd gitops

            # Branch do GitOps DEVE existir, sem fallback
            git checkout "$GITOPS_BRANCH"
            git pull origin "$GITOPS_BRANCH"

            # Overlay por ambiente
            OVERLAY_DIR="k8s/deploy/$STAGE"
            [ -d "$OVERLAY_DIR" ] || { echo "Overlay nao encontrado: $OVERLAY_DIR"; exit 1; }

            # kustomization.(yaml|yml)
            KFILE="$OVERLAY_DIR/kustomization.yaml"
            [ -f "$OVERLAY_DIR/kustomization.yml" ] && KFILE="$OVERLAY_DIR/kustomization.yml"

            cd "$OVERLAY_DIR"
            # Substitui placeholder -> imagem+tag corretas
            kustomize edit set image "$AWS_ECR_URI/placeholder=$AWS_ECR_URI/$IMAGE_NAME:$FULL_SHA"

            echo "==== $(basename "$KFILE") após edição ===="
            cat "$(basename "$KFILE")"

            # sanity check: não pode sobrar ':latest'
            if kustomize build . | grep -q ":latest"; then
              echo "ERRO: imagem ficou :latest no render"
              exit 1
            fi
            cd - >/dev/null

            # Commit/push apenas se mudou
            if ! git diff --quiet -- "$KFILE"; then
              git add "$KFILE"
              git commit -m "[skip ci] ${STAGE}: Atualiza imagem para $AWS_ECR_URI/$IMAGE_NAME:$FULL_SHA"
              git push origin "$GITOPS_BRANCH"
            else
              echo "Nenhuma mudança para commitar"
            fi

            # ArgoCD sync
            if argocd login "$ARGOCD_SERVER:443" --insecure --grpc-web --auth-token "$ARGOCD_TOKEN"; then
              echo "Logado com token"
            else
              echo "Tentando login admin/senha…"
              argocd login "$ARGOCD_SERVER:443" --insecure --grpc-web --username admin --password "$ARGOCD_TOKEN"
            fi

            argocd app sync "$ARGOCD_APP_NAME" --force --prune --grpc-web
          '''
        }
      }
    }
  }

  post {
    always { echo 'Pipeline finalizada.' }
  }
}
