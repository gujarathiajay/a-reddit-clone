pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node 16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        APP_NAME = "reddit-clone-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "gujarathiajaykumar"
        DOCKER_PASS = 'dockerhub'
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        JENKINS_API_TOKEN = credentials('JENKINS_API_TOKEN')
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/gujarathiajay/a-reddit-clone.git'
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Reddit-Clone-CI \
                    -Dsonar.projectKey=Reddit-Clone-CI'''
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'SonarQube-Token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('', DOCKER_PASS) {
                        docker_image = docker.build("${IMAGE_NAME}")
                    }
                    docker.withRegistry('', DOCKER_PASS) {
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }

        stage("Trivy Image Scan") {
            steps {
                script {
                    sh '''
                    docker run -v /var/run/docker.sock:/var/run/docker.sock \
                    aquasec/trivy image gujarathiajaykumar/reddit-clone-pipeline:latest \
                    --no-progress --scanners vuln --exit-code 0 \
                    --severity HIGH,CRITICAL --format table > trivyimage.txt
                    '''
                }
            }
        }

        stage('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                    sh "docker rmi ${IMAGE_NAME}:latest || true"
                }
            }
        }

        stage("Trigger CD Pipeline") {
            steps {
                script {
                    // Secure API Token injection
                    withCredentials([string(credentialsId: 'JENKINS_API_TOKEN', variable: 'API_TOKEN')]) {
                        sh """
                        curl -v -k --user clouduser:${API_TOKEN} -X POST \
                        -H 'cache-control: no-cache' \
                        -H 'content-type: application/x-www-form-urlencoded' \
                        --data 'IMAGE_TAG=${IMAGE_TAG}' \
                        'http://ec2-65-2-187-142.ap-south-1.compute.amazonaws.com:8080/job/Reddit-Clone-CD/buildWithParameters?token=gitops-token'
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            emailext(
                attachLog: true,
                subject: "'${currentBuild.result}' - ${env.JOB_NAME} Build #${env.BUILD_NUMBER}",
                body: """
                    <b>Project:</b> ${env.JOB_NAME}<br/>
                    <b>Build Number:</b> ${env.BUILD_NUMBER}<br/>
                    <b>Result:</b> ${currentBuild.result}<br/>
                    <b>URL:</b> ${env.BUILD_URL}<br/>
                """,
                to: 'gujarathiajaykumar@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
        }
    }
}
