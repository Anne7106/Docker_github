pipeline {
agent any

environment {
    DOCKER_IMAGE = 'catherine7/online-bookstore:v2'
    DEPLOYMENT_NAME = 'online-bookstore'
    SERVICE_NAME = 'online-bookstore-service'
}

options {
    timestamps()
}

stages {

    stage('Clone Code') {
        steps {
            git branch: 'main',
                credentialsId: '111',
                url: 'https://github.com/Anne7106/Docker_github.git'
        }
    }

    stage('Build Docker Image') {
        steps {
            bat 'docker build -t %DOCKER_IMAGE% .'
        }
    }

    stage('Test Docker Image') {
        steps {
            bat 'docker image inspect %DOCKER_IMAGE%'
        }
    }

    stage('Push to Docker Hub') {
        steps {
            withCredentials([
                usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_TOKEN'
                )
            ]) {
                bat '''
                    echo %DH_TOKEN% | docker login -u %DH_USER% --password-stdin
                    docker push %DOCKER_IMAGE%
                    docker logout
                '''
            }
        }
    }

    stage('Deploy to Kubernetes') {
        steps {
            withCredentials([
                file(
                    credentialsId: 'kubeconfig',
                    variable: 'KUBECONFIG'
                )
            ]) {
                bat 'kubectl apply -f deployment.yaml'
                bat 'kubectl apply -f service.yaml'
                bat 'kubectl rollout status deployment/online-bookstore --timeout=120s'
            }
        }
    }

    stage('Verify Deployment') {
        steps {
            withCredentials([
                file(
                    credentialsId: 'kubeconfig',
                    variable: 'KUBECONFIG'
                )
            ]) {
                bat 'kubectl get deployment online-bookstore'
                bat 'kubectl get pods'
                bat 'kubectl get service online-bookstore-service'
                bat 'kubectl rollout status deployment/online-bookstore --timeout=120s'
            }
        }
    }
}

post {
    success {
        echo 'Online Bookstore CI/CD pipeline completed successfully!'
    }
    failure {
        echo 'Pipeline failed. Check Console Output for details.'
    }
    always {
        echo 'Online Bookstore pipeline execution finished.'
    }
}

}