
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    environment {
        DOCKER_EXE = 'C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe'
        KUBECTL_EXE = 'C:\\Program Files\\Docker\\Docker\\resources\\bin\\kubectl.exe'

        IMAGE_NAME = 'balaji00002/online-hospital-appointment-portal'
        IMAGE_TAG = 'v1'

        DEPLOYMENT_NAME = 'online-hospital-appointment-portal'
        SERVICE_NAME = 'hospital-portal-service'
        CONTAINER_NAME = 'hospital-portal'
        NODE_PORT = '30085'
    }

    stages {
        stage('1. Clone Code') {
            steps {
                echo 'Cloning the Hospital Appointment Portal from GitHub'
                checkout scm
            }
        }

        stage('2. Build Docker Image') {
            steps {
                bat '''
                    "%DOCKER_EXE%" build --pull=false -t %IMAGE_NAME%:%IMAGE_TAG% .
                '''
            }
        }

        stage('3. Test Application') {
            steps {
                bat '''
                    "%DOCKER_EXE%" image inspect %IMAGE_NAME%:%IMAGE_TAG%
                '''
            }
        }

        stage('4. Push Image') {
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
                        "%DOCKER_EXE%" login -u "%DOCKER_USERNAME%" --password-stdin < nul
                    '''
                    powershell '''
                        $ErrorActionPreference = "Stop"
                        $env:DOCKER_PASSWORD | & $env:DOCKER_EXE login `
                            --username $env:DOCKER_USERNAME `
                            --password-stdin
                        if ($LASTEXITCODE -ne 0) {
                            exit $LASTEXITCODE
                        }
                    '''
                    bat '''
                        "%DOCKER_EXE%" push %IMAGE_NAME%:%IMAGE_TAG%
                    '''
                }
            }
        }

        stage('5. Check Kubernetes') {
            steps {
                bat '''
                    "%KUBECTL_EXE%" config current-context
                    "%KUBECTL_EXE%" get nodes
                '''
            }
        }

        stage('6. Deploy to Kubernetes') {
            steps {
                script {
                    try {
                        bat '''
                            "%KUBECTL_EXE%" apply -f deployment.yaml
                        '''

                        bat '''
                            "%KUBECTL_EXE%" set image deployment/%DEPLOYMENT_NAME% %CONTAINER_NAME%=%IMAGE_NAME%:%IMAGE_TAG%
                        '''

                        bat '''
                            "%KUBECTL_EXE%" rollout status deployment/%DEPLOYMENT_NAME% --timeout=120s
                        '''
                    } catch (err) {
                        echo 'Deployment failed. Attempting rollback to the previous revision.'

                        bat '''
                            "%KUBECTL_EXE%" rollout undo deployment/%DEPLOYMENT_NAME%
                        '''

                        bat '''
                            "%KUBECTL_EXE%" rollout status deployment/%DEPLOYMENT_NAME% --timeout=120s
                        '''

                        error('New deployment failed. Rollback was attempted; inspect the rollout and Pods.')
                    }
                }
            }
        }

        stage('7. Verify Deployment') {
            steps {
                bat '''
                    "%KUBECTL_EXE%" get deployment %DEPLOYMENT_NAME%
                    "%KUBECTL_EXE%" get pods -l app=hospital-portal
                    "%KUBECTL_EXE%" get service %SERVICE_NAME%
                '''

                echo 'Hospital portal URL: http://localhost:30085'
            }
        }
    }

    post {
        success {
            echo 'Hospital Appointment Portal pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check Console Output and Kubernetes rollout status.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
