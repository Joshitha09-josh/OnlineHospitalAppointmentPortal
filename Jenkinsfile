
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'balaji00002/online-hospital-appointment-portal'
        STABLE_TAG = 'v1'
        DOCKER = 'C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe'
        KUBECTL = 'C:\\Program Files\\Docker\\Docker\\resources\\bin\\kubectl.exe'
        DEPLOYMENT_NAME = 'online-hospital-appointment-portal'
        SERVICE_NAME = 'hospital-portal-service'
    }

    parameters {
        choice(
            name: 'RELEASE_MODE',
            choices: ['STABLE', 'TEST_FAULTY_RELEASE'],
            description: 'STABLE deploys v1. TEST_FAULTY_RELEASE tests automatic rollback.'
        )
    }

    stages {
        stage('1. Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('2. Build Docker Image') {
            steps {
                bat '"%DOCKER%" build -t %DOCKER_IMAGE%:%STABLE_TAG% .'
            }
        }

        stage('3. Test Application Image') {
            steps {
                bat '"%DOCKER%" image inspect %DOCKER_IMAGE%:%STABLE_TAG%'
            }
        }

        stage('4. Push Stable Image') {
            when {
                expression { params.RELEASE_MODE == 'STABLE' }
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    bat '''
                        @echo off
                        echo %DOCKER_PASSWORD% | "%DOCKER%" login -u "%DOCKER_USERNAME%" --password-stdin
                        if errorlevel 1 exit /b 1
                        "%DOCKER%" push %DOCKER_IMAGE%:%STABLE_TAG%
                        if errorlevel 1 exit /b 1
                        "%DOCKER%" logout
                    '''
                }
            }
        }

        stage('5. Save Current Stable Release') {
            steps {
                bat '''
                    @echo off
                    "%KUBECTL%" get deployment %DEPLOYMENT_NAME% -o yaml > previous-deployment.yaml
                    if errorlevel 1 exit /b 1
                    "%KUBECTL%" get deployment %DEPLOYMENT_NAME% -o jsonpath="{.spec.template.spec.containers[0].image}" > previous-image.txt
                    if errorlevel 1 exit /b 1
                    type previous-image.txt
                '''
            }
        }

        stage('6. Deploy Release') {
            steps {
                script {
                    if (params.RELEASE_MODE == 'TEST_FAULTY_RELEASE') {
                        bat '''
                            "%KUBECTL%" set image deployment/%DEPLOYMENT_NAME% hospital-portal=%DOCKER_IMAGE%:faulty
                        '''
                    } else {
                        bat '''
                            "%KUBECTL%" apply -f deployment.yaml
                            if errorlevel 1 exit /b 1
                            "%KUBECTL%" set image deployment/%DEPLOYMENT_NAME% hospital-portal=%DOCKER_IMAGE%:%STABLE_TAG%
                            if errorlevel 1 exit /b 1
                        '''
                    }
                }
            }
        }

        stage('7. Verify Rollout and Roll Back on Failure') {
            steps {
                script {
                    int rolloutResult = bat(
                        returnStatus: true,
                        script: '"%KUBECTL%" rollout status deployment/%DEPLOYMENT_NAME% --timeout=90s'
                    )

                    if (rolloutResult != 0) {
                        echo 'Rollout failed. Restoring the previous stable Kubernetes revision.'
                        bat '"%KUBECTL%" rollout undo deployment/%DEPLOYMENT_NAME%'
                        bat '"%KUBECTL%" rollout status deployment/%DEPLOYMENT_NAME% --timeout=120s'
                    } else {
                        echo 'Deployment rollout completed successfully.'
                    }
                }
            }
        }

        stage('8. Verify Application') {
            steps {
                bat '"%KUBECTL%" get deployments'
                bat '"%KUBECTL%" get pods -l app=hospital-portal'
                bat '"%KUBECTL%" get services'
                bat '"%KUBECTL%" rollout history deployment/%DEPLOYMENT_NAME%'
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed. Check rollout and Pod status in Console Output.'
        }
        failure {
            echo 'Pipeline failed. Inspect Console Output and Kubernetes rollout status.'
        }
    }
}
