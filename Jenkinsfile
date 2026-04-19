pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '2'))
        timestamps()
    }

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool('sonar-scanner')
        DOCKER_IMAGE = 'sylvesteryiadom/boardgame:latest'
        K8S_SERVER_URL = 'https://4A5C0DFAEA555F1C802190CB4D7C4D49.gr7.us-east-1.eks.amazonaws.com'
        K8S_CLUSTER_NAME = 'devops-cluster'
        K8S_NAMESPACE = 'webapps'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/sylvesteryiadom/Boardgame.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('File System Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.txt .'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=BoardGame \
                        -Dsonar.projectKey=BoardGame \
                        -Dsonar.java.binaries=.
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }

        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3') {
                    sh 'mvn deploy'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE .'
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh 'trivy image --format table -o trivy-image-report.txt $DOCKER_IMAGE'
            }
        }

        stage('Push Docker Image') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred', url: '') {
                    sh 'docker push $DOCKER_IMAGE'
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig([
                    credentialsId: 'k8-cred',
                    serverUrl: env.K8S_SERVER_URL,
                    clusterName: env.K8S_CLUSTER_NAME,
                    namespace: env.K8S_NAMESPACE,
                    restrictKubeConfigAccess: true
                ]) {
                    sh 'kubectl apply -f deployment-service.yaml'
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig([
                    credentialsId: 'k8-cred',
                    serverUrl: env.K8S_SERVER_URL,
                    clusterName: env.K8S_CLUSTER_NAME,
                    namespace: env.K8S_NAMESPACE,
                    restrictKubeConfigAccess: true
                ]) {
                    sh 'kubectl get pods -n $K8S_NAMESPACE'
                    sh 'kubectl get svc -n $K8S_NAMESPACE'
                }
            }
        }
    }
}
