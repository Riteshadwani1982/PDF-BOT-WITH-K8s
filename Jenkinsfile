pipeline {
    agent any
    options { timestamps(); disableConcurrentBuilds() }

    environment {
        AWS_REGION = 'us-east-1' // change to your existing region
        AWS_ACCOUNT_ID = '692859921402'
        ECR_REPOSITORY = 'pdf-bot'
        EKS_CLUSTER = 'CHANGE_ME_EKS_CLUSTER'
        HELM_RELEASE = 'pdf-bot'
        CHART_PATH = 'helm/pdf-bot'
    }

    stages {
        stage('Select environment') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        env.TARGET_ENV = 'prod'
                        env.K8S_NAMESPACE = 'pdf-bot-prod'
                    } else if (env.BRANCH_NAME == 'Development') {
                        env.TARGET_ENV = 'dev'
                        env.K8S_NAMESPACE = 'pdf-bot-dev'
                    } else {
                        error("No EKS deployment configured for branch ${env.BRANCH_NAME}")
                    }
                    env.ECR_URI = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/${env.ECR_REPOSITORY}"
                }
            }
        }

        stage('Checkout and validate') {
            steps {
                checkout scm
                sh '''
                    set -eu
                    aws --version
                    docker --version
                    kubectl version --client
                    helm version
                    helm lint "$CHART_PATH"
                '''
            }
        }

        stage('Build and push image') {
            steps {
                script {
                    env.COMMIT_SHA = sh(script: 'git rev-parse --short=12 HEAD', returnStdout: true).trim()
                    env.IMAGE_TAG = "${env.TARGET_ENV}-${env.BUILD_NUMBER}-${env.COMMIT_SHA}"
                    env.IMAGE_URI = "${env.ECR_URI}:${env.IMAGE_TAG}"
                }
                sh '''
                    set -eu
                    aws ecr get-login-password --region "$AWS_REGION" |
                      docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
                    docker build --pull -t "$IMAGE_URI" .
                    docker push "$IMAGE_URI"
                '''
            }
        }

        stage('Deploy with Helm to EKS') {
            steps {
                withCredentials([string(credentialsId: 'google-api-key', variable: 'GOOGLE_API_KEY')]) {
                    sh '''
                        set -eu
                        umask 077
                        printf '%s' "$GOOGLE_API_KEY" > "$WORKSPACE/google-api-key.tmp"

                        aws eks update-kubeconfig --region "$AWS_REGION" --name "$EKS_CLUSTER"

                        helm upgrade --install "$HELM_RELEASE" "$CHART_PATH"                           --namespace "$K8S_NAMESPACE" --create-namespace                           --set image.repository="$ECR_URI"                           --set image.tag="$IMAGE_TAG"                           --set-file secret.googleApiKey="$WORKSPACE/google-api-key.tmp"                           --wait --timeout 10m

                        kubectl rollout status deployment/"$HELM_RELEASE"                           -n "$K8S_NAMESPACE" --timeout=300s
                        kubectl get deploy,pods,svc,ingress,hpa,configmap,secret -n "$K8S_NAMESPACE"
                    '''
                }
            }
        }

        stage('Endpoint details') {
            steps {
                sh '''
                    kubectl get ingress "$HELM_RELEASE" -n "$K8S_NAMESPACE"
                    echo "After the ALB address appears, test http://<ALB-DNS>/demo"
                '''
            }
        }
    }

    post {
        always {
            sh 'rm -f google-api-key.tmp || true'
            echo "Pipeline result: ${currentBuild.currentResult}"
        }
    }
}
