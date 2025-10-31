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

    stage('Push Image to Azure Container Registry') {
        steps {
            withCredentials([azureServicePrincipal(credentialsId: 'AZURE_CREDENTIALS')]) {
                script {
                    sh '''
                        # Login to Azure using Service Principal
                        az login --service-principal \
                            -u $AZURE_CLIENT_ID \
                            -p $AZURE_CLIENT_SECRET \
                            --tenant $AZURE_TENANT_ID
    
                        # Login to ACR
                        az acr login --name hardkacr
    
                        # Tag and push Docker images
                        IMAGE_TAG=${GIT_COMMIT}
                        docker tag getting-started:$IMAGE_TAG hardkacr.azurecr.io/getting-started:$IMAGE_TAG
                        docker tag getting-started:$IMAGE_TAG hardkacr.azurecr.io/getting-started:latest
                        docker push hardkacr.azurecr.io/getting-started:$IMAGE_TAG
                        docker push hardkacr.azurecr.io/getting-started:latest
                    '''
                }
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

