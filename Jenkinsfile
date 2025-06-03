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

                    EKS_CLUSTER_NAME="eks-store"
                    AWS_REGION="sa-east-1"
                    KUBECONFIG_TEMP_PATH="/tmp/kubeconfig-eks-store.yaml"
                    PROMETHEUS_YAML_PATH="prometheus/k8s/k8s.yaml" # Caminho relativo ao seu workspace Jenkins

                    echo "--- Gerando kubeconfig inicial ---"
                    aws eks update-kubeconfig \\
                        --name "${EKS_CLUSTER_NAME}" \\
                        --region "${AWS_REGION}" \\
                        --kubeconfig "${KUBECONFIG_TEMP_PATH}"

                    echo "--- Modificando username no kubeconfig temporário ---"
                    sed -i "s|name: arn:aws:eks:${AWS_REGION}:730335608828:cluster/${EKS_CLUSTER_NAME}|name: guilherme.kaidei|g" "${KUBECONFIG_TEMP_PATH}"
                    sed -i "s|user: arn:aws:eks:${AWS_REGION}:730335608828:cluster/${EKS_CLUSTER_NAME}|user: guilherme.kaidei|g" "${KUBECONFIG_TEMP_PATH}"

                    echo "--- Conteúdo do kubeconfig modificado (para depuração) ---"
                    cat "${KUBECONFIG_TEMP_PATH}"

                    echo "--- Verificando permissões com o kubeconfig modificado ---"
                    kubectl --kubeconfig "${KUBECONFIG_TEMP_PATH}" auth can-i create clusterroles
                    kubectl --kubeconfig "${KUBECONFIG_TEMP_PATH}" auth can-i create clusterrolebindings
                    kubectl --kubeconfig "${KUBECONFIG_TEMP_PATH}" auth can-i get clusterroles
                    kubectl --kubeconfig "${KUBECONFIG_TEMP_PATH}" auth can-i get configmaps -n kube-system

                    echo "--- Aplicando o YAML do Prometheus ao cluster EKS ---"
                    kubectl --kubeconfig "${KUBECONFIG_TEMP_PATH}" apply -f "${PROMETHEUS_YAML_PATH}"

                    echo "--- Verificação final dos pods do Prometheus ---"
                    kubectl --kubeconfig "${KUBECONFIG_TEMP_PATH}" get pods -l app=prometheus
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
