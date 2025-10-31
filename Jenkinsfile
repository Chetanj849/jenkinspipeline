pipeline {
    agent any

    environment {
        DOCKER_REPO = "yourdockerhubuser/getting-started"
        K8S_PATH = "k8s/overlays/dev" // change to prod when needed
    }

    stages {
       stage('Checkout Source') {
            steps {
                git(
                    url: 'https://github.com/Chetanj849/getting-started.git',
                    branch: 'master',
                )
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                    docker build -t ${DOCKER_REPO}:${GIT_COMMIT} .
                    """
                }
            }
        }

        stage('Push Image to Registry') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh '''
                    echo "$PASS" | docker login -u "$USER" --password-stdin
                    docker push ${DOCKER_REPO}:${GIT_COMMIT}
                    '''
                }
            }
        }

        stage('Prepare Kustomize') {
            steps {
                // Clone your deployment config repo (where Kustomize lives)
                dir('deployment-config') {
                    git url: 'https://github.com/youruser/docker-getting-started-deploy.git', branch: 'main'

                    dir("${K8S_PATH}") {
                        sh """
                        kustomize edit set image ${DOCKER_REPO}=${DOCKER_REPO}:${GIT_COMMIT}
                        git config user.name "jenkins"
                        git config user.email "jenkins@ci.local"
                        git commit -am "Update image to ${GIT_COMMIT}"
                        git push origin main
                        """
                    }
                }
            }
        }

        stage('Deploy to AKS') {
            steps {
                withKubeConfig([credentialsId: 'aks-kubeconfig']) {
                    dir('deployment-config') {
                        sh "kubectl apply -k ${K8S_PATH}"
                    }
                }
            }
        }
    }
}

