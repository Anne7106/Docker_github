
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    parameters {
        booleanParam(
            name: 'SIMULATE_FAILURE',
            defaultValue: false,
            description: 'Deploy a faulty image tag to test automatic rollback'
        )
    }

    environment {
        DOCKER_IMAGE = 'catherine7/student-placement-portal'
        APP_NAME = 'student-placement-portal'
        CONTAINER_NAME = 'student-placement-portal'
    }

    stages {
        stage('Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t %DOCKER_IMAGE%:%BUILD_NUMBER% .'
            }
        }

        stage('Test Docker Image') {
            steps {
                bat 'docker image inspect %DOCKER_IMAGE%:%BUILD_NUMBER%'
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
                        @echo off
                        echo %DH_TOKEN%| docker login -u %DH_USER% --password-stdin
                        if errorlevel 1 exit /b 1
                        docker push %DOCKER_IMAGE%:%BUILD_NUMBER%
                        if errorlevel 1 exit /b 1
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
    steps {
        withCredentials([
            file(
                credentialsId: 'kubeconfig-credentials',
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
                        credentialsId: 'kubeconfig-credentials',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    bat '''
                        kubectl rollout status deployment/student-placement-portal --timeout=120s
                        kubectl get deployment student-placement-portal
                        kubectl get pods -l app=student-placement-portal
                        kubectl get service student-placement-service
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Student Placement Portal pipeline completed.'
        }
    }
}
