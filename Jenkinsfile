pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: jnlp
      image: jenkins/inbound-agent:latest
      args: ['\$(JENKINS_SECRET)', '\$(JENKINS_NAME)']
      tty: true
    - name: docker
      image: docker:24.0.5-dind
      securityContext:
        privileged: true
      env:
        - name: DOCKER_TLS_CERTDIR
          value: ""
      volumeMounts:
        - name: docker-graph-storage
          mountPath: /var/lib/docker
  volumes:
    - name: docker-graph-storage
      emptyDir: {}
  restartPolicy: Never
"""
            defaultContainer 'jnlp'
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

        stage('Build & Push Docker Image') {
            steps {
                container('docker') {
                    script {
                        def imageTag = "jonatandvs/frontend:${env.BUILD_NUMBER}"
                        sh "docker build -t ${imageTag} ."
                        sh "docker login -u ${DOCKERHUB_CREDENTIALS_USR} -p ${DOCKERHUB_CREDENTIALS_PSW}"
                        sh "docker push ${imageTag}"
                        env.IMAGE_TAG = imageTag
                    }
                }
            }
        }

        stage('Update Helm Chart') {
            steps {
                container('docker') {
                    script {
                        sh "git clone https://github.com/JonatnV/ChartTemplate.git charts-repo"
                        sh """
                            yq e '.frontend.image = "jonatandvs/frontend"' -i charts-repo/charts/app/values-dev.yaml
                            yq e '.frontend.tag = "${BUILD_NUMBER}"' -i charts-repo/charts/app/values-dev.yaml
                        """
                    }
                }
            }
        }

        stage('Package Helm Chart') {
            steps {
                container('docker') {
                    sh "helm package charts-repo/charts/app -d charts-repo/packages"
                }
            }
        }

        stage('Upload to ChartMuseum') {
            steps {
                container('docker') {
                    sh """
                        curl --user ${CHARTMUSEUM_CREDENTIALS_USR}:${CHARTMUSEUM_CREDENTIALS_PSW} \
                        --data-binary @charts-repo/packages/webapp-0.1.0.tgz \
                        ${CHART_REPO}/api/charts
                    """
                }
            }
        }

        stage('Cleanup') {
            steps {
                container('docker') {
                    sh "rm -rf charts-repo"
                }
            }
        }
    }
}
