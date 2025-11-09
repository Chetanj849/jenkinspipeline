pipeline {
    agent any

    environment {
        // --- Repositories ---
        APP_REPO = "https://github.com/Chetanj849/getting-started.git"
        APP_BRANCH = "master"

        INFRA_REPO = "https://github.com/Chetanj849/jenkinspipeline.git"
        K8S_OVERLAY = "infra/k8s/overlays/dev"

        // --- Azure Details ---
        TENANT_ID = 'e4e34038-ea1f-4882-b6e8-ccd776459ca0'
        ACR_NAME = 'hardkacr'
        AKS_RG = 'hardik-rg'
        AKS_NAME = 'aks-1'

        // --- Image Info ---
        IMAGE_NAME = 'docker-getting-started'
        IMAGE_TAG = "v${env.BUILD_NUMBER}"
    }

    stages {

        // -------------------------------------------------------------------
        stage('Checkout Source Repositories') {
            steps {
                script {
                    echo "📦 Checking out repositories..."
                    dir('app') {
                        git branch: "${APP_BRANCH}", url: "${APP_REPO}"
                    }
                    dir('infra') {
                        git branch: "master", url: "${INFRA_REPO}"
                    }
                }
            }
        }

        // -------------------------------------------------------------------
        stage('Build Docker Image') {
            steps {
                dir('app') {
                    script {
                        sh '''
                        echo "🛠️ Building Docker image..."
                        docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                        '''
                    }
                }
            }
        }

        // -------------------------------------------------------------------
        stage('Install Tools') {
            steps {
                sh '''
                echo "🔧 Installing Azure CLI..."
                curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

                echo "🔧 Installing kubectl..."
                curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                chmod +x kubectl
                sudo mv kubectl /usr/local/bin/kubectl

                echo "✅ Installed tools (Azure CLI + kubectl):"
                az --version
                kubectl version --client
                '''
            }
        }

        // -------------------------------------------------------------------
        stage('Push Image to Azure Container Registry') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'azure-sp',
                        usernameVariable: 'AZURE_CLIENT_ID',
                        passwordVariable: 'AZURE_CLIENT_SECRET'
                    )
                ]) {

                    sh '''
                    echo "🔐 Logging into Azure..."
                    az login --service-principal \
                        --username "$AZURE_CLIENT_ID" \
                        --password "$AZURE_CLIENT_SECRET" \
                        --tenant "'"${TENANT_ID}"'"

                    echo "🔓 Logging into ACR..."
                    az acr login --name '${ACR_NAME}'

                    echo "📤 Tagging and pushing Docker images..."
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:${IMAGE_TAG}

                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:latest
                    docker push ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:latest

                    echo "✅ Pushed images to ACR successfully"
                    '''
                }
            }
        }

        // -------------------------------------------------------------------
        stage('Prepare Kustomize Config') {
            steps {
                dir('infra') {
                    script {
                        sh '''
                        echo "🔧 Installing Kustomize..."
                        curl -s https://api.github.com/repos/kubernetes-sigs/kustomize/releases/latest \
                          | grep browser_download_url \
                          | grep linux_amd64.tar.gz \
                          | cut -d '"' -f 4 | wget -qi -

                        tar -xzf kustomize_*_linux_amd64.tar.gz
                        chmod +x kustomize

                        echo "🖊️ Updating image reference in Kustomize..."
                        cd k8s/overlays/dev
                        ../../../kustomize edit set image ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:latest
                        '''
                    }
                }
            }
        }

        // -------------------------------------------------------------------
        stage('Deploy to AKS') {
            steps {
                withCredentials([file(credentialsId: 'aks-kubeconfig-file', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                    echo "📡 Deploying to AKS..."
                    export KUBECONFIG=$KUBECONFIG_FILE

                    kubectl apply -k infra/k8s/overlays/dev

                    echo "✅ Deployment applied successfully!"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Check the logs above."
        }
    }
}
