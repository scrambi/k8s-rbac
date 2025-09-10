pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
  }

  environment {
    NS         = 'gitlab-admin'                // namespace для SA
    SA_NAME    = 'gitlab-ci-admin'             // имя сервис-аккаунта
    KCFG_OUT   = 'kubeconfig-gitlab-ci-admin'  // имя генерируемого kubeconfig
    TG_CHAT_ID = '180424264'                   // твой chat_id
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Apply RBAC') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            set -euo pipefail
            kubectl apply -f rbac/00-namespace.yaml
            kubectl apply -f rbac/10-sa.yaml
            kubectl apply -f rbac/20-crb.yaml
          '''
        }
      }
    }

    stage('Issue token & build kubeconfig') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            set -euo pipefail
            # не светим секреты в логе
            set +x

            # URL API сервера и CA (берём из kubeconfig; CA уже base64)
            SERVER="$(kubectl config view --raw -o jsonpath='{.clusters[0].cluster.server}')"
            CA_B64="$(kubectl config view  --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')"

            # Токен: новый способ (1.24+) + fallback для старых кластеров
            if TOKEN="$(kubectl -n ${NS} create token ${SA_NAME} --duration=87600h 2>/dev/null)"; then
              :
            else
              kubectl -n ${NS} apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: ${SA_NAME}-token
  namespace: ${NS}
  annotations:
    kubernetes.io/service-account.name: "${SA_NAME}"
type: kubernetes.io/service-account-token
EOF
              # подождём, пока контроллер наполнит secret
              for i in {1..10}; do
                TOKEN="$(kubectl -n ${NS} get secret ${SA_NAME}-token -o jsonpath='{.data.token}' 2>/dev/null || true)"
                [ -n "$TOKEN" ] && { TOKEN="$(echo "$TOKEN" | base64 -d)"; break; }
                sleep 1
              done
            fi

            cat > "${KCFG_OUT}" <<EOF
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: ${CA_B64}
    server: ${SERVER}
  name: k8s
contexts:
- context:
    cluster: k8s
    namespace: ${NS}
    user: ${SA_NAME}
  name: ${SA_NAME}@k8s
current-context: ${SA_NAME}@k8s
users:
- name: ${SA_NAME}
  user:
    token: ${TOKEN}
EOF
          '''
        }
      }
    }

    stage('Publish kubeconfig') {
      steps {
        archiveArtifacts artifacts: "${KCFG_OUT}", fingerprint: true
      }
    }
  }

  post {
    success {
      withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'TG')]) {
        sh '''
          set +x
          TG_CLEAN=$(printf %s "$TG" | tr -d '\\r\\n')
          # sanity-check (не роняем билд)
          curl -fsS "https://api.telegram.org/bot$TG_CLEAN/getMe" -o /dev/null || true
          # notify
          curl -fsS -X POST "https://api.telegram.org/bot$TG_CLEAN/sendMessage" \
            -d "chat_id='${TG_CHAT_ID}'" \
            --data-urlencode "text=✅ SA создан/обновлён: ${JOB_NAME} #${BUILD_NUMBER}" \
            -o /dev/null || true
        '''
      }
    }
    failure {
      withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'TG')]) {
        sh '''
          set +x
          TG_CLEAN=$(printf %s "$TG" | tr -d '\\r\\n')
          curl -fsS -X POST "https://api.telegram.org/bot$TG_CLEAN/sendMessage" \
            -d "chat_id='${TG_CHAT_ID}'" \
            --data-urlencode "text=❌ Ошибка: ${JOB_NAME} #${BUILD_NUMBER}. Проверь логи Jenkins." \
            -o /dev/null || true
        '''
      }
    }
  }
}
