pipeline {
    agent {
        kubernetes {
            label 'ci-frontend-kaniko'
            defaultContainer 'jnlp'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    command:
      - /busybox/sh
      - -c
      - "tail -f /dev/null"
    volumeMounts:
      - name: kaniko-secret
        mountPath: /kaniko/.docker
  volumes:
  - name: kaniko-secret
    secret:
      secretName: regcred
"""
        }
    }

    environment {
        CHARTMUSEUM_CREDENTIALS = credentials('chartmuseum-credentials')
        CHART_REPO = 'http://10.118.0.4:32080'
        CHART_NAME = 'webapp'
        IMAGE_NAME = 'jonatandvs/frontend'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/JonatnV/Frontend.git', credentialsId: 'github-credentials'
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    def imageTag = "${IMAGE_NAME}:${BUILD_NUMBER}"
                    env.IMAGE_TAG = imageTag
                    container('kaniko') {
                        sh """
                        /kaniko/executor \
                            --dockerfile=$WORKSPACE/Dockerfile \
                            --context=dir://$WORKSPACE \
                            --destination=${imageTag} \
                            --skip-tls-verify
                        """
                    }
                }
            }
        }

        stage('Update Helm Chart') {
            steps {
                script {
                    sh "git clone https://github.com/JonatnV/ChartTemplate.git charts-repo"
                    sh """
                        yq e '.frontend.image = "${IMAGE_NAME}"' -i charts-repo/charts/app/values-dev.yaml
                        yq e '.frontend.tag = "${BUILD_NUMBER}"' -i charts-repo/charts/app/values-dev.yaml
                    """
                }
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
                    --data-binary @charts-repo/packages/${CHART_NAME}-0.1.0.tgz \
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
