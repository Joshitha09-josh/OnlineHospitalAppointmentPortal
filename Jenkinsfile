
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'balaji00002/online-hospital-appointment-portal'
        DOCKER_TAG = 'v1'
        DOCKER = 'C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe'
        KUBECTL = 'C:\\Program Files\\Docker\\Docker\\resources\\bin\\kubectl.exe'
        DEPLOYMENT_NAME = 'online-hospital-appointment-portal'
    }

    stages {
        stage('1. Clone Code') {
            steps {
                echo 'Cloning hospital appointment portal from GitHub'
                checkout scm
            }
        }

        stage('2. Build Docker Image') {
            steps {
                bat '"%DOCKER%" build --pull -t %DOCKER_IMAGE%:%DOCKER_TAG% .'
            }
        }

        stage('3. Test Application') {
            steps {
                bat '"%DOCKER%" image inspect %DOCKER_IMAGE%:%DOCKER_TAG%'
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
                        echo %DOCKER_PASSWORD% | "%DOCKER%" login -u "%DOCKER_USERNAME%" --password-stdin
                        if errorlevel 1 exit /b 1
                        "%DOCKER%" push %DOCKER_IMAGE%:%DOCKER_TAG%
                        if errorlevel 1 exit /b 1
                        "%DOCKER%" logout
                    '''
                }
            }
        }

        stage('5. Check Kubernetes') {
            steps {
                bat '"%KUBECTL%" config current-context'
                bat '"%KUBECTL%" get nodes'
            }
        }

        stage('6. Deploy to Kubernetes') {
            steps {
                bat '"%KUBECTL%" apply -f deployment.yaml'
                bat '"%KUBECTL%" rollout status deployment/%DEPLOYMENT_NAME% --timeout=120s'
            }
        }

        stage('7. Verify Deployment') {
            steps {
                bat '"%KUBECTL%" get deployments'
                bat '"%KUBECTL%" get pods'
                bat '"%KUBECTL%" get services'
            }
        }
    }

    post {
        success {
            echo 'Hospital appointment portal CI/CD pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the first failed stage in Console Output.'
        }
    }
}
