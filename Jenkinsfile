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
                    # Substitua 'eks-store' e 'sa-east-1' pelos valores corretos do seu ambiente
                    EKS_CLUSTER_NAME="eks-store"
                    AWS_REGION="sa-east-1"
                    KUBECONFIG_TEMP_PATH="/tmp/kubeconfig-eks-store.yaml"
                    PROMETHEUS_YAML_PATH="prometheus/k8s/k8s.yaml" # Caminho para o seu arquivo YAML do Prometheus
                    
                    # O nome padrão do usuário/contexto gerado por aws eks update-kubeconfig
                    # Ele é o ARN completo do cluster
                    DEFAULT_KUBECONFIG_NAME="arn:aws:eks:${AWS_REGION}:730335608828:cluster/${EKS_CLUSTER_NAME}"
                    TARGET_USERNAME="guilherme.kaidei" # O username mapeado no seu aws-auth ConfigMap
                    
                    echo "--- Gerando kubeconfig inicial ---"
                    # 1. Gerar o kubeconfig para um arquivo temporário.
                    #    Ele virá com o ARN do cluster como username/context name por padrão.
                    aws eks update-kubeconfig \
                        --name "${EKS_CLUSTER_NAME}" \
                        --region "${AWS_REGION}" \
                        --kubeconfig "${KUBECONFIG_TEMP_PATH}"
                    
                    echo "--- Modificando kubeconfig com comandos kubectl config ---"
                    # 2. Definir a variável de ambiente KUBECONFIG para que todos os comandos kubectl usem este arquivo.
                    export KUBECONFIG="${KUBECONFIG_TEMP_PATH}"
                    
                    # Renomear a entrada do usuário no kubeconfig
                    # Isso muda o 'name' na seção 'users:' de 'arn:aws:eks:...' para 'guilherme.kaidei'
                    kubectl config rename-user "${DEFAULT_KUBECONFIG_NAME}" "${TARGET_USERNAME}"
                    
                    # Atualizar o contexto para usar o novo nome de usuário
                    # O nome do contexto ainda é o ARN completo, mas agora ele aponta para o usuário renomeado.
                    kubectl config set-context "${DEFAULT_KUBECONFIG_NAME}" --user="${TARGET_USERNAME}"
                    
                    # Opcional: Renomear o contexto para 'guilherme.kaidei' para consistência
                    # Isso muda o 'name' na seção 'contexts:'
                    kubectl config rename-context "${DEFAULT_KUBECONFIG_NAME}" "${TARGET_USERNAME}"
                    
                    # Definir o contexto atual para o novo nome (guilherme.kaidei)
                    kubectl config use-context "${TARGET_USERNAME}"
                    
                    echo "--- Conteúdo do kubeconfig modificado (para depuração) ---"
                    # 3. Exibir o conteúdo do kubeconfig modificado para confirmar as alterações.
                    cat "${KUBECONFIG_TEMP_PATH}"
                    
                    echo "--- Verificando permissões com o kubeconfig modificado ---"
                    # 4. Verifique as permissões. Agora, todos devem retornar 'yes'.
                    kubectl auth can-i create clusterroles
                    kubectl auth can-i create clusterrolebindings
                    kubectl auth can-i get clusterroles
                    kubectl auth can-i get configmaps -n kube-system # Este pode ainda ser 'no', mas não é o bloqueador
                    
                    echo "--- Aplicando o YAML do Prometheus ao cluster EKS ---"
                    # 5. Aplicar o YAML do Prometheus.
                    #    Como o kubeconfig está configurado corretamente, isso deve funcionar.
                    kubectl apply -f "${PROMETHEUS_YAML_PATH}"
                    
                    echo "--- Verificação final dos pods do Prometheus ---"
                    # 6. Verificar o status dos pods do Prometheus.
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
