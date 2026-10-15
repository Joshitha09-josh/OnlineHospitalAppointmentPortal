```groovy
pipeline {
    agent any

    environment {
        DOCKER = 'C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe'
        KUBECTL = 'C:\\Program Files\\Docker\\Docker\\resources\\bin\\kubectl.exe'
        IMAGE = 'balaji00002/hospital-appointment-portal'
        TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '"C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe" build -t balaji00002/hospital-appointment-portal:%BUILD_NUMBER% .'
            }
        }

        stage('Test Application') {
            steps {
                bat '"C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe" image inspect balaji00002/hospital-appointment-portal:%BUILD_NUMBER%'
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    bat '''
                    "@C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe" login -u "%DOCKER_USERNAME%" --password-stdin < nul
                    '''
                    bat '''
                    echo %DOCKER_PASSWORD% | "C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe" login -u "%DOCKER_USERNAME%" --password-stdin
                    '''
                    bat '"C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe" tag balaji00002/hospital-appointment-portal:%BUILD_NUMBER% balaji00002/hospital-appointment-portal:latest'
                    bat '"C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe" push balaji00002/hospital-appointment-portal:%BUILD_NUMBER%'
                    bat '"C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe" push balaji00002/hospital-appointment-portal:latest'
                    bat '"C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe" logout'
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    try {
                        bat '"C:\\Program Files\\Docker\\Docker\\resources\\bin\\kubectl.exe" set image deployment/hospital-appointment-portal hospital-appointment-portal=balaji00002/hospital-appointment-portal:%BUILD_NUMBER%'
                        bat '"C:\\Program Files\\Docker\\Docker\\resources\\bin\\kubectl.exe" rollout status deployment/hospital-appointment-portal --timeout=120s'
                    } catch (err) {
                        bat '"C:\\Program Files\\Docker\\Docker\\resources\\bin\\kubectl.exe" rollout undo deployment/hospital-appointment-portal'
                        bat '"C:\\Program Files\\Docker\\Docker\\resources\\bin\\kubectl.exe" rollout status deployment/hospital-appointment-portal --timeout=120s'
                        error('Deployment failed. Rollback to the previous revision was attempted.')
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                bat '"C:\\Program Files\\Docker\\Docker\\resources\\bin\\kubectl.exe" get deployments'
                bat '"C:\\Program Files\\Docker\\Docker\\resources\\bin\\kubectl.exe" get pods'
                bat '"C:\\Program Files\\Docker\\Docker\\resources\\bin\\kubectl.exe" get services'
            }
        }
    }

    post {
        success {
            echo 'Hospital portal pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Review the first failed stage and rollback output.'
        }
    }
}
```