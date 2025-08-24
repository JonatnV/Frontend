pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    command:
      - /busybox/sh
    args:
      - -c
      - "sleep 99d"   # Mantener el contenedor vivo mientras Jenkins ejecuta sh
"""
        }
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        CHARTMUSEUM_CREDENTIALS = credentials('chartmuseum-credentials')
        CHART_REPO = 'http://10.118.0.4:32080'
        CHART_NAME = 'webapp'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/JonatnV/Frontend.git', credentialsId: 'github-credentials'
            }
        }

        stage('Build Docker Image') {
            steps {
                container('kaniko') {
                    script {
                        def imageTag = "jonatandvs/frontend:${env.BUILD_NUMBER}"
                        sh """
                        /kaniko/executor \
                          --dockerfile=$WORKSPACE/Dockerfile \
                          --context=dir://$WORKSPACE \
                          --destination=${imageTag} \
                          --skip-tls-verify
                        """
                        env.IMAGE_TAG = imageTag
                    }
                }
            }
        }

        stage('Update Helm Chart') {
            steps {
                sh "git clone https://github.com/JonatnV/ChartTemplate.git charts-repo"
                sh """
                  yq e '.frontend.image = "jonatandvs/frontend"' -i charts-repo/charts/app/values-dev.yaml
                  yq e '.frontend.tag = "${BUILD_NUMBER}"' -i charts-repo/charts/app/values-dev.yaml
                """
            }
        }

        stage('Package Helm Chart') {
            steps {
                sh "helm package charts-repo/charts/app -d charts-repo/packages"
            }
        }

        stage('Upload to ChartMuseum') {
            steps {
                sh """
                  curl --user ${CHARTMUSEUM_CREDENTIALS_USR}:${CHARTMUSEUM_CREDENTIALS_PSW} \
                  --data-binary @charts-repo/packages/webapp-0.1.0.tgz \
                  ${CHART_REPO}/api/charts
                """
            }
        }

        stage('Cleanup') {
            steps {
                sh "rm -rf charts-repo"
            }
        }
    }
}
