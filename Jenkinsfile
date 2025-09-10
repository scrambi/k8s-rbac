pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
  }

  environment {
    NS         = 'gitlab-admin'           // namespace для SA
    SA_NAME    = 'gitlab-ci-admin'        // имя сервис-аккаунта
    KCFG_OUT   = 'kubeconfig-gitlab-ci-admin' // имя генерируемого kubeconfig
    TG_CHAT_ID = '180424264'              // <<-- твой chat_id
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Apply RBAC') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            set -e
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
            set -e
            # не светим секреты в логе
            set +x

            # URL API сервера и CA (берём прямо из kubeconfig; уже base64)
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
              TOKEN="$(kubectl -n ${NS} get secret ${SA_NAME}-token -o jsonpath='{.data.token}' | base64 -d)"
            fi

            cat > ${KCFG_OUT} <<EOF
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
        archiveArtifacts artifacts: 'kubeconfig-gitlab-ci-admin', fingerprint: true
      }
    }
  }

  post {
    success {
      withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'TG')]) {
        sh '''
          curl -s -X POST -H 'Content-Type: application/json' \
            --data-binary @- https://api.telegram.org/bot'"${TG}"'/sendMessage <<EOF
{"chat_id":"'"${TG_CHAT_ID}"'","text":"✅ SA *gitlab-ci-admin* создан, kubeconfig сгенерирован (job: '"${JOB_NAME}"' #'"${BUILD_NUMBER}"').","parse_mode":"Markdown"}
EOF
        '''
      }
    }
    failure {
      withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'TG')]) {
        sh '''
          curl -s -X POST -H 'Content-Type: application/json' \
            --data-binary @- https://api.telegram.org/bot'"${TG}"'/sendMessage <<EOF
{"chat_id":"'"${TG_CHAT_ID}"'","text":"❌ Ошибка при создании SA для GitLab (job: '"${JOB_NAME}"' #'"${BUILD_NUMBER}"'). См. логи Jenkins."}
EOF
        ''' || true
      }
    }
  }
}
