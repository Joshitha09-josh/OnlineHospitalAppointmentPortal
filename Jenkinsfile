
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'balaji00002/online-hospital-appointment-portal'
        RELEASE_TAG = "build-${BUILD_NUMBER}"

        DOCKER = 'C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe'
        KUBECTL = 'C:\\Program Files\\Docker\\Docker\\resources\\bin\\kubectl.exe'

        DEPLOYMENT_NAME = 'online-hospital-appointment-portal'
        SERVICE_NAME = 'hospital-portal-service'
    }

    parameters {
        choice(
            name: 'RELEASE_MODE',
            choices: ['STABLE', 'TEST_FAULTY_RELEASE'],
            description: 'STABLE deploys a new image. TEST_FAULTY_RELEASE tests automatic rollback.'
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
                bat '"%DOCKER%" build -t %DOCKER_IMAGE%:%RELEASE_TAG% .'
            }
        }

        stage('3. Test Application Image') {
            steps {
                bat '"%DOCKER%" image inspect %DOCKER_IMAGE%:%RELEASE_TAG%'
            }
        }

        stage('4. Push Image to Docker Hub') {
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

                        "%DOCKER%" push %DOCKER_IMAGE%:%RELEASE_TAG%
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

                    echo Previously deployed image:
                    type previous-image.txt
                '''
            }
        }

        stage('6. Deploy Release') {
            steps {
                script {
                    if (params.RELEASE_MODE == 'TEST_FAULTY_RELEASE') {
                        echo 'Testing a deliberately unavailable image tag.'

                        bat '''
                            "%KUBECTL%" set image deployment/%DEPLOYMENT_NAME% hospital-portal=%DOCKER_IMAGE%:faulty-do-not-exist
                            if errorlevel 1 exit /b 1
                        '''
                    } else {
                        bat '''
                            "%KUBECTL%" apply -f deployment.yaml
                            if errorlevel 1 exit /b 1

                            "%KUBECTL%" set image deployment/%DEPLOYMENT_NAME% hospital-portal=%DOCKER_IMAGE%:%RELEASE_TAG%
                            if errorlevel 1 exit /b 1
                        '''
                    }
                }
            }
        }

        stage('7. Verify Rollout and Automatic Rollback') {
            steps {
                script {
                    int rolloutResult = bat(
                        returnStatus: true,
                        script: '"%KUBECTL%" rollout status deployment/%DEPLOYMENT_NAME% --timeout=90s'
                    )

                    if (rolloutResult != 0) {
                        echo 'ROLLOUT FAILED. Rolling back to the previous Kubernetes revision.'

                        bat '"%KUBECTL%" rollout undo deployment/%DEPLOYMENT_NAME%'
                        bat '"%KUBECTL%" rollout status deployment/%DEPLOYMENT_NAME% --timeout=120s'

                        echo 'Rollback completed. The previous deployment is available again.'
                    } else {
                        echo 'Rollout succeeded.'
                    }
                }
            }
        }

        stage('8. Verify Deployment, Pods and Service') {
            steps {
                bat '"%KUBECTL%" get deployment %DEPLOYMENT_NAME%'
                bat '"%KUBECTL%" get pods -l app=hospital-portal'
                bat '"%KUBECTL%" get service %SERVICE_NAME%'
                bat '"%KUBECTL%" rollout history deployment/%DEPLOYMENT_NAME%'
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully. Check the rollout and Pod status.'
        }
        failure {
            echo 'Pipeline failed. Check Console Output and Kubernetes rollout status.'
        }
    }
}
