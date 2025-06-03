pipeline {
    agent any
    environment {
        SERVICE = 'bottleneck'
        NAME = "guikaidei/${env.SERVICE}"
    }
    stages {
        stage('Deploy Prometheus') {
            steps {
                sh '''
                    #!/bin/bash

                    # Define variáveis para o nome do cluster e região
                    EKS_CLUSTER_NAME="eks-store"
                    AWS_REGION="sa-east-1"
                    KUBECONFIG_TEMP_PATH="/tmp/kubeconfig-eks-store.yaml"
                    PROMETHEUS_YAML_PATH="prometheus/k8s/k8s.yaml" # Caminho para o seu arquivo YAML do Prometheus
                    
                    TARGET_USERNAME="guilherme.kaidei" # O username mapeado no seu aws-auth ConfigMap
                    
                    echo "--- Instalando dependências (jq) ---"
                    # Instalar jq se não estiver presente.
                    # Use 'apt-get' para sistemas baseados em Debian/Ubuntu, ou 'yum' para sistemas baseados em RHEL/CentOS.
                    # Certifique-se de que o usuário do Jenkins tem permissões para instalar pacotes (sudo).
                    # Se o agente Jenkins for um container, você pode precisar adicionar 'jq' à imagem do container.
                    if ! command -v jq &> /dev/null
                    then
                        echo "jq não encontrado. Tentando instalar..."
                        # Para Debian/Ubuntu:
                        sudo apt-get update && sudo apt-get install -y jq
                        # Para RHEL/CentOS:
                        # sudo yum install -y jq
                    else
                        echo "jq já está instalado."
                    fi
                    
                    
                    echo "--- Construindo kubeconfig manualmente ---"
                    
                    # 1. Obter detalhes do cluster EKS (endpoint e CA data)
                    #    Requer 'jq' para parsear a saída JSON
                    CLUSTER_INFO=$(aws eks describe-cluster --name "${EKS_CLUSTER_NAME}" --region "${AWS_REGION}" --query 'cluster.{endpoint:endpoint,caData:certificateAuthority.data}' --output json)
                    CLUSTER_ENDPOINT=$(echo "${CLUSTER_INFO}" | jq -r '.endpoint')
                    CLUSTER_CA_DATA=$(echo "${CLUSTER_INFO}" | jq -r '.caData')
                    
                    # 2. Obter o token de autenticação para o usuário atual
                    #    Isso usará a identidade IAM que o Jenkins está usando (guilherme.kaidei via SSO)
                    #    Requer 'jq' para parsear a saída JSON
                    AUTH_TOKEN=$(aws eks get-token --cluster-name "${EKS_CLUSTER_NAME}" --region "${AWS_REGION}" --output json | jq -r '.status.token')
                    
                    # 3. Criar o arquivo kubeconfig manualmente usando um 'here-document'
                    #    Isso garante que o YAML seja formatado corretamente e contenha as informações essenciais.
                    cat <<EOF > "${KUBECONFIG_TEMP_PATH}"
                    apiVersion: v1
                    clusters:
                    - cluster:
                        certificate-authority-data: ${CLUSTER_CA_DATA}
                        server: ${CLUSTER_ENDPOINT}
                      name: ${EKS_CLUSTER_NAME} # Nome do cluster no kubeconfig
                    contexts:
                    - context:
                        cluster: ${EKS_CLUSTER_NAME}
                        user: ${TARGET_USERNAME}
                      name: ${TARGET_USERNAME}@${EKS_CLUSTER_NAME} # Nome do contexto
                    current-context: ${TARGET_USERNAME}@${EKS_CLUSTER_NAME}
                    kind: Config
                    preferences: {}
                    users:
                    - name: ${TARGET_USERNAME}
                      user:
                        token: ${AUTH_TOKEN}
                    EOF
                    
                    echo "--- Conteúdo do kubeconfig construído (para depuração) ---"
                    # 4. Exibir o conteúdo do kubeconfig construído para confirmar as alterações.
                    cat "${KUBECONFIG_TEMP_PATH}"
                    
                    # 5. Definir a variável de ambiente KUBECONFIG para que todos os comandos kubectl usem este arquivo.
                    export KUBECONFIG="${KUBECONFIG_TEMP_PATH}"
                    
                    echo "--- Verificando permissões com o kubeconfig construído ---"
                    # 6. Verifique as permissões. Agora, todos devem retornar 'yes'.
                    #    Se 'jq' não estiver instalado, os comandos acima falharão antes de chegar aqui.
                    kubectl auth can-i create clusterroles
                    kubectl auth can-i create clusterrolebindings
                    kubectl auth can-i get clusterroles
                    kubectl auth can-i get configmaps -n kube-system # Este pode ainda ser 'no', mas não é o bloqueador
                    
                    echo "--- Aplicando o YAML do Prometheus ao cluster EKS ---"
                    # 7. Aplicar o YAML do Prometheus.
                    #    Como o kubeconfig está configurado corretamente, isso deve funcionar.
                    kubectl apply -f "${PROMETHEUS_YAML_PATH}"
                    
                    echo "--- Verificação final dos pods do Prometheus ---"
                    # 8. Verificar o status dos pods do Prometheus.
                    kubectl get pods -l app=prometheus
                    
                    # Opcional: Desdefinir a variável KUBECONFIG para não afetar passos subsequentes do pipeline
                    unset KUBECONFIG
                    
                    # Opcional: Limpar o arquivo kubeconfig temporário após o uso
                    # rm "${KUBECONFIG_TEMP_PATH}"

                '''
            }
        }

        stage('Deploy grafana to Kubernetes') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials'
                ]]) {
                    sh '''
                        aws eks update-kubeconfig --name eks-store --region sa-east-1
                        kubectl apply -f grafana/k8s/k8s.yaml
                    '''
                }
            }
        }

        stage('Deploy redis to Kubernetes') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials'
                ]]) {
                    sh '''
                        aws eks update-kubeconfig --name eks-store --region sa-east-1
                        kubectl apply -f redis/k8s/k8s.yaml
                    '''
                }
            }
        }
        
    }
}
