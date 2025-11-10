
pipeline {
    agent any

    environment {
        // --- Base ---
        DOCKER_REPO = "getting-started"
        K8S_PATH = "k8s/overlays/dev"

        // --- Azure Details ---
        AZURE_CREDENTIALS = credentials('azure-sp')
        TENANT_ID = 'e4e34038-ea1f-4882-b6e8-ccd776459ca0'
        ACR_NAME = 'acr849'
        AKS_RG = 'Chetan-RG'
        AKS_NAME = 'aks-1'

        // --- Image Info ---
        IMAGE_NAME = 'docker-getting-started'
        IMAGE_TAG = "v${env.BUILD_NUMBER}"
    }

    stages {
        // -------------------------------

        stage('Checkout Source') { 
            steps { 
                git( url: 'https://github.com/Chetanj849/getting-started.git', branch: 'master' ) 
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
                        sh """
                            az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant e4e34038-ea1f-4882-b6e8-ccd776459ca0
                            az acr login --name ${ACR_NAME}
        
                            docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:${IMAGE_TAG}
                            docker push ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:${IMAGE_TAG}
        
                            docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:latest
                            docker push ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:latest
                        """
                    }
                }
            }
        }


        // -------------------------------
        stage('Prepare Kustomize') {
            steps {
                script {
                    echo "🔧 Installing Kustomize locally..."
                    sh '''
                        cd $WORKSPACE
                        curl -s https://api.github.com/repos/kubernetes-sigs/kustomize/releases/latest \
                        | grep browser_download_url | grep linux_amd64.tar.gz | cut -d '"' -f 4 | wget -qi -
                        tar -xzf kustomize_v*_linux_amd64.tar.gz
                        chmod +x kustomize
        
                        echo "🔧 Updating image tag in Kustomize..."
                        cd k8s/overlays/dev
                        ../../../kustomize edit set image hardkacr.azurecr.io/docker-getting-started:latest
                    '''
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
