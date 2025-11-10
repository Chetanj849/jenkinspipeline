pipeline {
    agent any
 
    environment {
        // --- Repositories ---
        APP_REPO = "https://github.com/Chetanj849/getting-started.git"
        APP_BRANCH = "master"
 
        // --- Azure Details ---
        AZURE_CREDENTIALS = credentials('azure-sp')
        TENANT_ID = 'e4e34038-ea1f-4882-b6e8-ccd776459ca0'
        ACR_NAME = 'acr84'
        AKS_RG = 'hardik-rg'
        AKS_NAME = 'aks-1'
 
        // --- Image Info ---
        IMAGE_NAME = 'docker-getting-started'
        IMAGE_TAG = "v${env.BUILD_NUMBER}"
    }
 
    stages {
 
        // -------------------------------
        stage('Checkout Source Repos') {
            steps {
                script {
                    echo "📦 Checking out repositories..."
                    dir('app') {
                        git branch: "${APP_BRANCH}", url: "${APP_REPO}"
                    }
                    dir('infra') {
                        git branch: "master", url: "https://github.com/Chetanj849/jenkinspipeline.git"
                    }
                }
            }
        }
 
        // -------------------------------
        stage('Build Docker Image') {
            steps {
                dir('app') {
                    script {
                        sh """
                            echo "🛠️ Building Docker image..."
                            docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                        """
                    }
                }
            }
        }
 
        // -------------------------------
        stage('Push Image to ACR') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'azure-sp', usernameVariable: 'AZURE_CLIENT_ID', passwordVariable: 'AZURE_CLIENT_SECRET')]) {
                    script {
                        sh """
                            echo "🔐 Logging into Azure..."
                            az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant ${TENANT_ID}
                            az acr login --name ${ACR_NAME}
 
                            echo "📤 Pushing Docker image..."
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
                dir('infra') {
                    script {
                        sh '''
                            echo "🔧 Installing Kustomize..."
                            curl -s https://api.github.com/repos/kubernetes-sigs/kustomize/releases/latest \
                              | grep browser_download_url | grep linux_amd64.tar.gz | cut -d '"' -f 4 | wget -qi -
                            tar -xzf kustomize_v*_linux_amd64.tar.gz
                            chmod +x kustomize
                        '''
                    }
                }
            }
        }
 
        // -------------------------------
        stage('Update Dev & Prod Image') {
            steps {
                dir('infra') {
                    script {
                        sh """
                            echo "🔧 Updating DEV overlay..."
                            cd k8s/overlays/dev
                            ../../../kustomize edit set image ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:${IMAGE_TAG}

                            echo "🔧 Updating PROD overlay..."
                            cd ../prod
                            ../../../kustomize edit set image ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:${IMAGE_TAG}
                        """
                    }
                }
            }
        }
 
        // -------------------------------
        stage('Install kubectl') {
            steps {
                sh '''
                echo "🔧 Installing kubectl..."
                curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                chmod +x kubectl
                mv kubectl /var/lib/jenkins/kubectl
                '''
            }
        }
 
        // -------------------------------
        stage('Deploy to DEV') {
            steps {
                withCredentials([file(credentialsId: 'aks-kubeconfig-file', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                    export KUBECONFIG=$KUBECONFIG_FILE
                    /var/lib/jenkins/kubectl apply -k infra/k8s/overlays/dev
                    '''
                }
            }
        }

        // -------------------------------
        stage('Deploy to PROD') {
            when {
                branch 'master'
            }
            steps {
                input message: "Proceed with PROD deployment?", ok: "Deploy"
                withCredentials([file(credentialsId: 'aks-kubeconfig-file', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                    export KUBECONFIG=$KUBECONFIG_FILE
                    /var/lib/jenkins/kubectl apply -k infra/k8s/overlays/prod
                    '''
                }
            }
        }
    }
 
    post {
        success {
            echo "✅ Dev & Prod Deployment Completed Successfully!"
        }
        failure {
            echo "❌ Pipeline Failed. Check logs."
        }
    }
}
