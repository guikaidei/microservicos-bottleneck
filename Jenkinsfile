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
                    #    Estes comandos são mais precisos para evitar duplicatas.
                    
                    # Modifica a entrada 'name' na seção 'users'
                    # Garante que a substituição ocorra apenas na linha que começa com '- name: ' e termina com o ARN completo
                    sed -i "s|^- name: ${EKS_CLUSTER_ARN}$|- name: ${TARGET_USERNAME}|" "${KUBECONFIG_TEMP_PATH}"
                    
                    # Modifica a entrada 'user' dentro do 'context'
                    # Garante que a substituição ocorra apenas na linha que começa com '  user: ' e termina com o ARN completo
                    sed -i "s|^  user: ${EKS_CLUSTER_ARN}$|  user: ${TARGET_USERNAME}|" "${KUBECONFIG_TEMP_PATH}"
                    
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
