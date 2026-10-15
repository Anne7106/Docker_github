
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    parameters {
        booleanParam(
            name: 'INJECT_FAILURE',
            defaultValue: false,
            description: 'Deploy a deliberately missing image to demonstrate automatic rollback'
        )
    }

    environment {
        IMAGE_NAME = 'catherine7/college-event-portal'
        APP_NAME = 'college-event-portal'
        SERVICE_NAME = 'college-event-portal-service'
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
                bat 'docker build -t %IMAGE_NAME%:%BUILD_NUMBER% .'
            }
        }

        stage('Test Application') {
            steps {
                bat 'docker image inspect %IMAGE_NAME%:%BUILD_NUMBER%'
            }
        }

        stage('Push Image') {
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
                        if errorlevel 1 exit /b 1

                        docker push %IMAGE_NAME%:%BUILD_NUMBER%
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
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    bat 'kubectl apply -f deployment.yaml'
                    bat 'kubectl apply -f service.yaml'
                    bat 'kubectl rollout status deployment/%APP_NAME% --timeout=120s'

                    script {
                        def deployImage = "${env.IMAGE_NAME}:${env.BUILD_NUMBER}"

                        if (params.INJECT_FAILURE) {
                            deployImage = "${env.IMAGE_NAME}:deliberately-missing"
                            echo 'Demonstration mode: deploying a nonexistent image tag.'
                        }

                        bat "kubectl set image deployment/${env.APP_NAME} event-portal=${deployImage}"
                    }
                }
            }
        }

        stage('Verify Rollout and Roll Back if Needed') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    script {
                        def rolloutResult = bat(
                            returnStatus: true,
                            script: 'kubectl rollout status deployment/%APP_NAME% --timeout=90s'
                        )

                        if (rolloutResult != 0) {
                            echo 'Rollout failed. Rolling back to the previous revision.'

                            bat 'kubectl rollout undo deployment/%APP_NAME%'
                            bat 'kubectl rollout status deployment/%APP_NAME% --timeout=120s'

                            if (params.INJECT_FAILURE) {
                                echo 'Faulty rollout detected and previous stable release restored.'
                            } else {
                                error('Deployment failed; rollback was attempted.')
                            }
                        }

                        bat 'kubectl get deployment %APP_NAME%'
                        bat 'kubectl get pods -l app=college-event-portal'
                        bat 'kubectl get service %SERVICE_NAME%'
                    }
                }
            }
        }

        stage('Verify Website') {
            when {
                expression { return !params.INJECT_FAILURE }
            }
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    bat 'kubectl get endpoints %SERVICE_NAME%'
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed. Check deployment status and website.'
        }
        failure {
            echo 'Pipeline failed. Inspect Console Output and Kubernetes rollout status.'
        }
        always {
            echo 'College Event Management pipeline finished.'
        }
    }
}
