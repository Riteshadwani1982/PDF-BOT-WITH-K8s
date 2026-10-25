pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION     = 'ap-southeast-1'
        AWS_ACCOUNT_ID = '692859921402'
        ECR_REPOSITORY = 'pdf-bot-repo'
        EKS_CLUSTER    = 'pdf-bot-eks-cluster'
        K8S_NAMESPACE  = 'default'
        HELM_RELEASE   = 'pdf-bot'
        CHART_PATH     = 'helm/pdf-bot'
        KUBECONFIG     = '/var/lib/jenkins/.kube/config'
    }

    stages {
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

                    kubectl get deployment "$HELM_RELEASE" \
                      -n "$K8S_NAMESPACE"

                    kubectl get secret pdf-bot-secret \
                      -n "$K8S_NAMESPACE"
                '''
            }
        }

        stage('Build and push image') {
            steps {
                script {
                    env.COMMIT_SHA = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()

                    env.ECR_URI = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/${env.ECR_REPOSITORY}"

                    env.IMAGE_TAG = "build-${env.BUILD_NUMBER}-${env.COMMIT_SHA}"
                    env.IMAGE_URI = "${env.ECR_URI}:${env.IMAGE_TAG}"
                }

                sh '''
                    set -eu

                    aws ecr get-login-password --region "$AWS_REGION" |
                      docker login \
                        --username AWS \
                        --password-stdin \
                        "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

                    docker build --pull -t "$IMAGE_URI" .
                    docker push "$IMAGE_URI"
                '''
            }
        }

        stage('Deploy with Helm to EKS') {
            steps {
                sh '''
                    set -eu

                    aws eks update-kubeconfig \
                      --region "$AWS_REGION" \
                      --name "$EKS_CLUSTER" \
                      --kubeconfig "$KUBECONFIG"

                    helm upgrade --install "$HELM_RELEASE" "$CHART_PATH" \
                      --namespace "$K8S_NAMESPACE" \
                      --set image.repository="$ECR_URI" \
                      --set image.tag="$IMAGE_TAG" \
                      --wait \
                      --timeout 10m

                    kubectl rollout status \
                      deployment/"$HELM_RELEASE" \
                      -n "$K8S_NAMESPACE" \
                      --timeout=300s

                    kubectl get deployment,pods,svc,ingress \
                      -n "$K8S_NAMESPACE"
                '''
            }
        }

        stage('Endpoint details') {
            steps {
                sh '''
                    set -eu
                    kubectl get ingress "$HELM_RELEASE" \
                      -n "$K8S_NAMESPACE"

                    echo "Chatbot endpoint:"
                    echo "http://k8s-default-pdfbot-97a6b9cb59-1659080412.ap-southeast-1.elb.amazonaws.com/demo"
                '''
            }
        }
    }

    post {
        always {
            echo "Pipeline result: ${currentBuild.currentResult}"
        }
    }
}
