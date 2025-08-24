pipeline {
    agent {
        kubernetes {
            label 'ci-frontend-kaniko'
            defaultContainer 'kaniko'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    args: ["--context=dir://workspace", "--dockerfile=Dockerfile", "--no-push"]
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
        IMAGE_NAME = 'jonatandvs/frontend'
        BUILD_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/JonatnV/Frontend.git'
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                container('kaniko') {
                    sh """
                    /kaniko/executor \
                        --dockerfile=$WORKSPACE/Dockerfile \
                        --context=dir://$WORKSPACE \
                        --destination=${IMAGE_NAME}:${BUILD_TAG} \
                        --skip-tls-verify
                    """
                }
            }
        }
    }
}
