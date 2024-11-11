pipeline {
    agent any

    environment {
        PROJECT_NAME = 'spring-boot-application'
        BUILD_NUMBER = "${env.BUILD_ID}"
        DOCKER_IMAGE = "my-docker-repo/${PROJECT_NAME}:${BUILD_NUMBER}"
        DOCKER_REGISTRY = 'my-docker-repo'
        STAGING_ENV = 'staging'
        PROD_ENV = 'production'
        SLACK_CHANNEL = '#devops-notifications'
    }

    options {
        timeout(time: 1, unit: 'HOURS')
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo 'Checking out source code'
                checkout scm
            }
        }

        stage('Code Quality & Security Scan') {
            parallel {
                stage('Static Code Analysis') {
                    steps {
                        echo 'Running static code analysis with SonarQube'
                        sh './gradlew sonarqube'
                    }
                }
                
                stage('Dependency Vulnerability Scan') {
                    steps {
                        echo 'Running dependency vulnerability scan'
                        sh './gradlew dependencyCheckAnalyze'
                    }
                }
            }
        }

        stage('Build & Unit Tests') {
            steps {
                echo 'Building project and running unit tests'
                sh './gradlew clean build'
            }
        }

        stage('Performance Tests') {
            steps {
                echo 'Running performance tests'
                sh './gradlew performanceTest'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image'
                script {
                    docker.build(DOCKER_IMAGE)
                }
            }
        }

        stage('Push Docker Image to Registry') {
            steps {
                echo 'Pushing Docker image to registry'
                script {
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-credentials') {
                        docker.image(DOCKER_IMAGE).push()
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                echo 'Deploying to staging environment'
                sh '''
                    kubectl set image deployment/${PROJECT_NAME} ${PROJECT_NAME}=${DOCKER_IMAGE} -n ${STAGING_ENV}
                '''
            }
        }

        stage('Integration Tests in Staging') {
            steps {
                echo 'Running integration tests in staging'
                sh './gradlew integrationTest'
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                echo 'Deploying to production environment'
                script {
                    try {
                        sh '''
                            kubectl set image deployment/${PROJECT_NAME} ${PROJECT_NAME}=${DOCKER_IMAGE} -n ${PROD_ENV}
                        '''
                    } catch (Exception e) {
                        echo 'Deployment to production failed. Rolling back'
                        sh '''
                            kubectl rollout undo deployment/${PROJECT_NAME} -n ${PROD_ENV}
                        '''
                        throw e
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up workspace'
            cleanWs()
        }

        success {
            echo 'Pipeline completed successfully.'
            slackSend(channel: SLACK_CHANNEL, color: 'good', message: "SUCCESS: Pipeline for ${PROJECT_NAME} #${BUILD_NUMBER} completed successfully.")
        }

        failure {
            echo 'Pipeline failed.'
            slackSend(channel: SLACK_CHANNEL, color: 'danger', message: "FAILURE: Pipeline for ${PROJECT_NAME} #${BUILD_NUMBER} failed. Please check Jenkins logs for details.")
        }

        unstable {
            echo 'Pipeline completed with issues.'
            slackSend(channel: SLACK_CHANNEL, color: 'warning', message: "UNSTABLE: Pipeline for ${PROJECT_NAME} #${BUILD_NUMBER} completed but with issues.")
        }
    }
}
