pipeline {
    agent any

    environment {
        // Base image repo
        IMAGE_NAME = "getting-started"
        IMAGE_TAG  = "v${env.BUILD_NUMBER}"

        // ACR
        ACR_NAME = "acr849"
        ACR_LOGIN_SERVER = "acr849.azurecr.io"

        // Kubernetes
        K8S_BASE = "k8s/overlays"

        // Deployment environment (dev or prod)
        DEPLOY_ENV = "dev"   // You can override this at build time
    }

    stages {

        stage('Checkout Source') {
            steps {
                git url: 'https://github.com/Chetanj849/jenkinspipeline.git', branch: 'master'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        echo 🔨 Building Docker image...
                        docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    """
                }
            }
        }

        stage('Login & Push to ACR') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'acr-admin-creds',
                        usernameVariable: 'ACR_USER',
                        passwordVariable: 'ACR_PASS'
                    )
                ]) {
                    sh """
                        echo 🔐 Logging into ACR...
                        echo $ACR_PASS | docker login ${ACR_LOGIN_SERVER} -u $ACR_USER --password-stdin

                        echo 📤 Pushing image to ACR...
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${ACR_LOGIN_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${ACR_LOGIN_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Prepare Kustomize') {
            steps {
                script {

                    echo "📦 Preparing Kustomize overlay for environment: ${DEPLOY_ENV}"

                    sh """
                        # Replace tag in patch.yaml for dev or prod
                        sed -i 's|NEW_IMAGE_TAG|${IMAGE_TAG}|g' ${K8S_BASE}/${DEPLOY_ENV}/patch.yaml

                        # Replace repo -> ACR repo
                        sed -i 's|IMAGE_REGISTRY|${ACR_LOGIN_SERVER}/${IMAGE_NAME}|g' ${K8S_BASE}/${DEPLOY_ENV}/patch.yaml
                    """
                }
            }
        }

        stage('Deploy to AKS') {
            steps {
                withCredentials([
                    file(credentialsId: 'aks-kubeconfig', variable: 'KUBECONFIG')
                ]) {
                    sh """
                        export KUBECONFIG=$KUBECONFIG
                        echo 🚀 Deploying to AKS using Kustomize...

                        kubectl apply -k ${K8S_BASE}/${DEPLOY_ENV}

                        echo ✅ Deployment Applied Successfully.
                    """
                }
            }
        }
    }

    post {
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
        success {
            echo "✅ Deployment successful for environment: ${DEPLOY_ENV}"
        }
    }
}
