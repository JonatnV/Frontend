pipeline {
    agent {
        kubernetes {
            label 'ci-frontend'
            defaultContainer 'jnlp'
            yamlFile 'jenkins-dind-pod.yaml' // Pod template Docker in Docker
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
                        sh "echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin"
                        sh "docker push ${imageTag}"
                        env.IMAGE_TAG = imageTag
                    }
                }
            }
        }

        stage('Update Helm Chart') {
            steps {
                script {
                    sh "git clone https://github.com/JonatnV/ChartTemplate.git charts-repo"
                    sh """
                        yq e '.frontend.image = "jonatandvs/frontend"' -i charts-repo/charts/app/values-dev.yaml
                        yq e '.frontend.tag = "${BUILD_NUMBER}"' -i charts-repo/charts/app/values-dev.yaml
                    """
                }
            }
        }

        stage('Package & Upload Helm Chart') {
            steps {
                container('docker') {
                    script {
                        sh "helm package charts-repo/charts/app -d charts-repo/packages"
                        sh """
                            curl --user ${CHARTMUSEUM_CREDENTIALS_USR}:${CHARTMUSEUM_CREDENTIALS_PSW} \
                            --data-binary @charts-repo/packages/${CHART_NAME}-0.1.0.tgz \
                            ${CHART_REPO}/api/charts
                        """
                    }
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh "rm -rf charts-repo"
            }
        }
    }
}
