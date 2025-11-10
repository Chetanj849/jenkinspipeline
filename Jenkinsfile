pipeline {
    agent any

    environment {
        IMAGE_NAME = "docker-getting-started"
        IMAGE_TAG  = "v${env.BUILD_NUMBER}"

        ACR_NAME = "acr84"
        TENANT_ID = "e4e34038-ea1f-4882-b6e8-ccd776459ca0"
    }

    stages {

        stage('Checkout Source') {
            steps {
                script {
                    echo "Cloning both repositories..."

                    dir('app') {
                        git url: 'https://github.com/Chetanj849/getting-started.git', branch: 'master'
                    }

                    dir('infra') {
                        git url: 'https://github.com/Chetanj849/jenkinspipeline.git', branch: 'master'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dir('app') {
                        sh """
                        docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                        """
                    }
                }
            }
        }

        stage('Push Image to ACR') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'azure-sp', usernameVariable: 'AZURE_CLIENT_ID', passwordVariable: 'AZURE_CLIENT_SECRET')
                ]) {
                    script {
                        sh """
                        az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant ${TENANT_ID}
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

        stage('Prepare Kustomize') {
            steps {
                script {
                    sh """
                    cd $WORKSPACE/infra
                    curl -s https://api.github.com/repos/kubernetes-sigs/kustomize/releases/latest \
                        | grep browser_download_url | grep linux_amd64.tar.gz | cut -d '"' -f 4 | wget -qi -
                    
                    tar -xzf kustomize_v*_linux_amd64.tar.gz
                    chmod +x kustomize

                    echo "Updating image in kustomize..."
                    cd k8s/overlays/dev
                    ../../../kustomize edit set image ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Deploy to AKS') {
            steps {
                sh """
                  az login --service-principal -u ${CLIENT_ID} -p ${CLIENT_SECRET} --tenant ${TENANT_ID}
                  az account set --subscription ${SUBSCRIPTION_ID}
                
                  az aks get-credentials \
                      --resource-group ${AKS_RG} \
                      --name ${AKS_NAME} \
                      --overwrite-existing
                
                  # Now kubectl works directly
                  kubectl apply -k k8s/overlays/dev
                """
                
                            }
                        }
                    }
                }
