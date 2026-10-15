
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
                    script {
                        bat '''
                            @echo off
                            kubectl get deployment %APP_NAME% >nul 2>&1
                            if errorlevel 1 (
                                kubectl apply -f deployment.yaml
                                if errorlevel 1 exit /b 1
                            )
                        '''

                        def imageTag = params.SIMULATE_FAILURE
                            ? 'faulty-does-not-exist'
                            : env.BUILD_NUMBER

                        bat """
                            kubectl set image deployment/${APP_NAME} ${CONTAINER_NAME}=${DOCKER_IMAGE}:${imageTag}
                        """

                        def result = bat(
                            returnStatus: true,
                            script: "kubectl rollout status deployment/${APP_NAME} --timeout=90s"
                        )

                        if (result != 0) {
                            echo 'Deployment failed. Rolling back to previous revision.'

                            bat """
                                kubectl rollout undo deployment/${APP_NAME}
                            """

                            def rollbackResult = bat(
                                returnStatus: true,
                                script: "kubectl rollout status deployment/${APP_NAME} --timeout=120s"
                            )

                            if (rollbackResult != 0) {
                                error('Rollback failed. Manual recovery is required.')
                            }

                            bat """
                                kubectl get deployment ${APP_NAME}
                                kubectl get pods -l app=${APP_NAME}
                                kubectl rollout history deployment/${APP_NAME}
                            """

                            error('Release failed, but the previous stable release was restored.')
                        }
                    }
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
