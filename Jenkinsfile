pipeline {
    agent any

    environment {
        // --- Base ---
        DOCKER_REPO = "getting-started"
        K8S_PATH = "k8s/overlays/dev"

        // --- Azure Details ---
        AZURE_CREDENTIALS = credentials('azure-sp')
        TENANT_ID = 'e4e34038-ea1f-4882-b6e8-ccd776459ca0'
        ACR_NAME = 'hardkacr'
        AKS_RG = 'hardik-rg'
        AKS_NAME = 'aks-1'

        // --- Image Info ---
        IMAGE_NAME = 'docker-getting-started'
        IMAGE_TAG = "v${env.BUILD_NUMBER}"
    }

    stages {
        // -------------------------------
        stage('Checkout Source') {
            steps {
                git(
                    url: 'https://github.com/Chetanj849/getting-started.git',
                    branch: 'master'
                )
            }
        }

        // -------------------------------
        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                    echo "🛠️ Building Docker image..."
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    """
                }
            }
        }

        // -------------------------------
        stage('Push Image to Azure Container Registry') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'azure-sp', usernameVariable: 'AZURE_CLIENT_ID', passwordVariable: 'AZURE_CLIENT_SECRET')]) {
                    script {
                        sh '''
                            az login --service-principal \
                                -u $AZURE_CLIENT_ID \
                                -p $AZURE_CLIENT_SECRET \
                                --tenant e4e34038-ea1f-4882-b6e8-ccd776459ca0
                
                            az acr login --name hardkacr
                
                            IMAGE_TAG=${GIT_COMMIT}
                            docker tag docker-getting-started:$IMAGE_TAG hardkacr.azurecr.io/docker-getting-started:$IMAGE_TAG
                            docker push hardkacr.azurecr.io/docker-getting-started:$IMAGE_TAG
                            docker tag docker-getting-started:$IMAGE_TAG hardkacr.azurecr.io/docker-getting-started:latest
                            docker push hardkacr.azurecr.io/docker-getting-started:latest
                        '''
                    }
                }

            }
        }


        // -------------------------------
        stage('Prepare Kustomize') {
            steps {
                dir('deployment-config') {
                    git url: 'https://github.com/Chetanj849/getting-started.git', branch: 'master'

                    dir("${K8S_PATH}") {
                        sh """
                        echo "🔧 Updating image tag in Kustomize..."
                        kustomize edit set image ${DOCKER_REPO}=${ACR_NAME}.azurecr.io/${IMAGE_NAME}:${IMAGE_TAG}

                        git config user.name "jenkins"
                        git config user.email "jenkins@ci.local"
                        git commit -am "Update image to ${IMAGE_TAG}"
                        git push origin main
                        """
                    }
                }
            }
        }

        // -------------------------------
        stage('Deploy to AKS') {
            steps {
                withKubeConfig([credentialsId: 'aks-kubeconfig']) {
                    dir('deployment-config') {
                        sh """
                        echo "🚀 Deploying to AKS..."
                        kubectl apply -k ${K8S_PATH}
                        """
                    }
                }
            }
        }
    }
}
