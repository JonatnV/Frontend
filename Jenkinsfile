pipeline {
    agent {
        kubernetes {
            label 'frontend-agent'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent
    args: ['\$(JENKINS_SECRET)', '\$(JENKINS_NAME)']
  - name: docker
    image: docker:24-dind
    securityContext:
      privileged: true
"""
        }
    }

    environment {
        GITHUB_CREDENTIALS = credentials('github_credential')
        DOCKERHUB_CREDENTIALS = credentials('dockerhub_credentials')
        CHARTMUSEUM_CREDENTIALS = credentials('chartmuseum_credentials')
        CHARTMUSEUM_URL = 'http://10.118.0.4:32080'  // Cambia por tu IP:puerto
    }

    stages {

        stage('Checkout Frontend') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/JonatnV/Frontend.git',
                    credentialsId: 'github_credential'
            }
        }

        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh """
                        docker build -t jonatanv/frontend:latest .
                        echo \$DOCKERHUB_CREDENTIALS_PSW | docker login -u \$DOCKERHUB_CREDENTIALS_USR --password-stdin
                        docker push jonatanv/frontend:latest
                    """
                }
            }
        }

        stage('Package Helm Chart') {
            steps {
                // Clonar template del chart
                git branch: 'main',
                    url: 'https://github.com/JonatnV/ChartTemplate.git',
                    credentialsId: 'github_credential'

                // Actualizar values.yaml con la nueva imagen
                sh """
                    sed -i 's|repository: .*|repository: jonatanv/frontend|' values.yaml
                    sed -i 's|tag: .*|tag: latest|' values.yaml
                """

                // Empaquetar chart
                sh 'helm package .'

                // Subir a ChartMuseum
                sh """
                    curl -u \$CHARTMUSEUM_CREDENTIALS_USR:\$CHARTMUSEUM_CREDENTIALS_PSW \
                         --data-binary @frontend-0.1.0.tgz \$CHARTMUSEUM_URL/api/charts
                """
            }
        }
    }
}
