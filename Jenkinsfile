pipeline {
    agent any
    environment {
        SERVICE = 'db'
        NAME = "guikaidei/${env.SERVICE}"
    }
    stages {
        stage('Deploy grafana to Kubernetes') {
            steps {
                sh 'kubectl apply -f grafana/k8s/k8s.yaml'
            }
        }

        stage('Deploy prometheus to Kubernetes') {
            steps {
                sh 'kubectl apply -f prometheus/k8s/k8s.yaml'
            }
        }

        stage('Deploy redis to Kubernetes') {
            steps {
                sh 'kubectl apply -f redis/k8s/k8s.yaml'
            }
        }
        
    }
}
