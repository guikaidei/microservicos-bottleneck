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
                    
                    # O ARN completo do cluster, conforme aparece no kubeconfig gerado
                    EKS_CLUSTER_ARN="arn:aws:eks:${AWS_REGION}:730335608828:cluster/${EKS_CLUSTER_NAME}"
                    TARGET_USERNAME="guilherme.kaidei"
                    
                    echo "--- Gerando kubeconfig inicial ---"
                    # 1. Gerar o kubeconfig inicial para um arquivo temporário.
                    #    Ele virá com o ARN do cluster como username por padrão.
                    aws eks update-kubeconfig \
                        --name "${EKS_CLUSTER_NAME}" \
                        --region "${AWS_REGION}" \
                        --kubeconfig "${KUBECONFIG_TEMP_PATH}"
                    
                    echo "--- Modificando username no kubeconfig temporário com sed preciso ---"
                    # 2. Modificar o username no arquivo kubeconfig gerado usando 'sed'.
                    #    Estes comandos são mais precisos para evitar duplicatas e alterações indesejadas.
                    
                    # Modifica o 'name' na seção 'users'.
                    # Procura pela linha que começa com '- name:' e contém o EKS_CLUSTER_ARN,
                    # e que está dentro do bloco 'users:'.
                    # Usa um padrão mais específico para evitar conflitos com 'name:' em outras seções.
                    sed -i "/^users:/,/^- name:/ { s|^- name: ${EKS_CLUSTER_ARN}$|- name: ${TARGET_USERNAME}|; t; }" "${KUBECONFIG_TEMP_PATH}"
                    
                    # Modifica o 'user' na seção 'contexts'.
                    # Procura pela linha que começa com '    user:' (com 4 espaços) e contém o EKS_CLUSTER_ARN,
                    # e que está dentro do bloco 'contexts:'.
                    sed -i "/^contexts:/,/^current-context:/ { s|^    user: ${EKS_CLUSTER_ARN}$|    user: ${TARGET_USERNAME}|; t; }" "${KUBECONFIG_TEMP_PATH}"
                    
                    # Modifica o 'name' do contexto atual (que também é o ARN do cluster por padrão)
                    # Isso é necessário porque o 'current-context' ainda aponta para o ARN completo
                    sed -i "s|^current-context: ${EKS_CLUSTER_ARN}$|current-context: ${TARGET_USERNAME}|" "${KUBECONFIG_TEMP_PATH}"
                    
                    # Modifica o 'name' do contexto dentro da lista de contextos
                    # Isso é para o caso de haver mais de um contexto, ou para garantir que o nome do contexto
                    # seja o mesmo do username que estamos usando.
                    sed -i "/^contexts:/,/^current-context:/ { s|^- name: ${EKS_CLUSTER_ARN}$|- name: ${TARGET_USERNAME}|; t; }" "${KUBECONFIG_TEMP_PATH}"
                    
                    
                    echo "--- Conteúdo do kubeconfig modificado (para depuração) ---"
                    cat "${KUBECONFIG_TEMP_PATH}"
                    
                    echo "--- Verificando permissões com o kubeconfig modificado ---"
                    # 3. Verifique as permissões com o kubeconfig modificado.
                    #    Estes comandos devem retornar 'yes' agora.
                    kubectl --kubeconfig "${KUBECONFIG_TEMP_PATH}" auth can-i create clusterroles
                    kubectl --kubeconfig "${KUBECONFIG_TEMP_PATH}" auth can-i create clusterrolebindings
                    kubectl --kubeconfig "${KUBECONFIG_TEMP_PATH}" auth can-i get clusterroles
                    kubectl --kubeconfig "${KUBECONFIG_TEMP_PATH}" auth can-i get configmaps -n kube-system # Este pode ainda ser 'no', mas não é o bloqueador
                    
                    echo "--- Aplicando o YAML do Prometheus ao cluster EKS ---"
                    # 4. Aplique o YAML do Prometheus usando o kubeconfig modificado.
                    #    Isso deve criar/atualizar todos os recursos do Prometheus (ConfigMap, ClusterRole, ClusterRoleBinding, Deployment, Service).
                    kubectl --kubeconfig "${KUBECONFIG_TEMP_PATH}" apply -f "${PROMETHEUS_YAML_PATH}"
                    
                    echo "--- Verificação final dos pods do Prometheus ---"
                    # 5. Verifique se o pod do Prometheus está rodando
                    kubectl --kubeconfig "${KUBECONFIG_TEMP_PATH}" get pods -l app=prometheus
                    
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
