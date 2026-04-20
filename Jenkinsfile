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
        REPO_URL = 'https://github.com/sylvesteryiadom/Boardgame.git'
        DOCKER_REPO = 'sylvesteryiadom/boardgame'
        DOCKER_IMAGE = "${DOCKER_REPO}:${BUILD_NUMBER}"
        K8S_SERVER_URL = 'https://029F6D62EE3A232D134A561F85E37062.gr7.us-east-1.eks.amazonaws.com'
        K8S_CLUSTER_NAME = 'devops-cluster'
        K8S_NAMESPACE = 'webapps'
        K8S_DEPLOYMENT = 'boardgame-deployment'
        MANIFEST = 'deployment-service.yaml'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: env.REPO_URL
            }
        }

        stage('Build And Test') {
            steps {
                sh 'mvn -B -ntp clean verify'
            }
        }

        stage('Scan And Analyze') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.txt .'
                withSonarQubeEnv('sonar') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=BoardGame \
                        -Dsonar.projectKey=BoardGame \
                        -Dsonar.java.binaries=target/classes
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
            }
        }

        stage('Publish Artifact') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3') {
                    sh 'mvn -B -ntp deploy -DskipTests'
                }
            }
        }

        stage('Build And Push Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE .'
                sh 'trivy image --format table -o trivy-image-report.txt $DOCKER_IMAGE'
                withDockerRegistry(credentialsId: 'docker-cred', url: '') {
                    sh 'docker push $DOCKER_IMAGE'
                    sh 'docker tag $DOCKER_IMAGE ${DOCKER_REPO}:latest'
                    sh 'docker push ${DOCKER_REPO}:latest'
                }
            }
        }

        stage('Deploy') {
            steps {
                withKubeConfig([
                    credentialsId: 'k8-cred',
                    serverUrl: env.K8S_SERVER_URL,
                    clusterName: env.K8S_CLUSTER_NAME,
                    namespace: env.K8S_NAMESPACE,
                    restrictKubeConfigAccess: false
                ]) {
                    sh '''
                        kubectl apply -f $MANIFEST
                        kubectl set image deployment/$K8S_DEPLOYMENT boardgame=$DOCKER_IMAGE -n $K8S_NAMESPACE
                        kubectl rollout status deployment/$K8S_DEPLOYMENT -n $K8S_NAMESPACE --timeout=180s
                        kubectl get svc -n $K8S_NAMESPACE
                    '''
                }
            }
        }
    }
}
