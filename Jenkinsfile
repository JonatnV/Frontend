pipeline {
    agent {
        kubernetes {
            label 'jenkins-agent-dind'
            defaultContainer 'jnlp'
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    some-label: jenkins-agent-dind
spec:
  containers:
  - name: docker
    image: docker:24-dind
    securityContext:
      privileged: true
    tty: true
    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
      type: Socket
"""
        }
    }
    environment {
        GITHUB_CREDENTIALS = credentials('github_credential')
        DOCKERHUB_CREDENTIALS = credentials('dockerhub_credentials')
        CHARTMUSEUM_CREDENTIALS = credentials('chartmuseum_credentials')
        CHARTMUSEUM_URL = 'http://10.118.0.4:32080'
        REPO_NAME = 'backend'
        IMAGE_NAME = 'backend'
        IMAGE_TAG = 'latest'
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/JonatnV/Backend.git',
                    credentialsId: "${GITHUB_CREDENTIALS}"
            }
        }
        stage('Build Docker Image') {
            steps {
                container('docker') {
                    script {
                        docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                    }
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                container('docker') {
                    script {
                        docker.withRegistry('', "${DOCKERHUB_CREDENTIALS}") {
                            docker.image("${IMAGE_NAME}:${IMAGE_TAG}").push()
                        }
                    }
                }
            }
        }
        stage('Update Helm Chart & Push') {
            steps {
                container('docker') {
                    script {
                        sh 'rm -rf chart && git clone https://github.com/JonatnV/ChartTemplate.git chart'
                        sh "sed -i 's|image: .*|image: ${IMAGE_NAME}:${IMAGE_TAG}|' chart/values.yaml"
                        sh "curl --user ${CHARTMUSEUM_CREDENTIALS_USR}:${CHARTMUSEUM_CREDENTIALS_PSW} --data-binary @chart -X POST ${CHARTMUSEUM_URL}/api/charts"
                    }
                }
            }
        }
    }
}
