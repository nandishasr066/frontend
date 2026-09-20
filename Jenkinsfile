pipeline {
    agent any

    environment {
        IMAGE_NAME = "nandishasrn/frontend:${GIT_COMMIT}"
    }

    stages {

        stage('Git Checkout') {
            steps {
                git(
                    url: 'https://github.com/nandishasr066/frontend.git',
                    branch: 'main'
                )
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh '''
                    go version
                    go test ./...
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME} .
                '''
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-creds',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin
                    '''
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh '''
                    docker push ${IMAGE_NAME}
                '''
            }
        }

        stage('Update GitOps Deployment') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-creds',
                    usernameVariable: 'GIT_USERNAME',
                    passwordVariable: 'GIT_PASSWORD'
                )]) {

                    sh '''
                        set -e

                        rm -rf gitops

                        cat > git-askpass.sh <<'EOF'
#!/bin/sh
case "$1" in
    *Username*)
        echo "$GIT_USERNAME"
        ;;
    *Password*)
        echo "$GIT_PASSWORD"
        ;;
esac
EOF

                        chmod 700 git-askpass.sh

                        export GIT_ASKPASS="$PWD/git-askpass.sh"
                        export GIT_TERMINAL_PROMPT=0

                        echo "Cloning GitOps repository..."

                        git clone \
                            https://github.com/nandishasr066/GitOps.git \
                            gitops

                        cd gitops/base/frontend

                        git config user.email "jenkins@ci.com"
                        git config user.name "jenkins"

                        echo "Current image:"
                        grep "image:" deployment.yaml

                        sed -i \
                            "s|image: .*frontend.*|image: ${IMAGE_NAME}|g" \
                            deployment.yaml

                        echo "Updated image:"
                        grep "image:" deployment.yaml

                        git add deployment.yaml

                        git commit \
                            -m "Update frontend image to ${IMAGE_NAME}" \
                            || echo "No changes to commit"

                        git push origin main

                        cd ../../..

                        rm -f git-askpass.sh
                    '''
                }
            }
        }
    }

    post {
        always {
            sh "docker rmi ${IMAGE_NAME} || true"
            sh "docker logout || true"
        }

        success {
            echo "Build and deployment update successful: ${IMAGE_NAME}"
        }

        failure {
            echo "Pipeline failed. Check the logs above."
        }
    }
}