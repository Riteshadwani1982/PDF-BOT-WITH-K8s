# PDF chatbot migration: ECR/ECS deployment to EKS + Helm + Jenkins

This bundle retains the supplied `app.py` and `requirements.txt`, adapts the Docker startup for Streamlit at `/demo`, adds Helm templates for Deployment, Service, Ingress, ConfigMap, Secret, and HPA, and replaces the ECS deployment stages in Jenkins with a Helm release to EKS.

## Before running
1. Copy this bundle's files into a branch of your existing repository. Keep your supplied `app.py` and `requirements.txt` at repository root; these are included unchanged.
2. Confirm `pdf-bot` ECR repository exists in the same account and region.
3. Update `AWS_REGION`, `AWS_ACCOUNT_ID` if needed, and `EKS_CLUSTER` in `Jenkinsfile`.
4. Ensure the Jenkins agent has AWS CLI, Docker, kubectl, Helm, and network connectivity to the EKS API.
5. Ensure the Jenkins AWS identity can push to ECR and deploy to the intended EKS namespaces. EKS access/RBAC is required in addition to IAM API permissions.
6. Add a Jenkins Secret Text credential with ID `google-api-key` containing the Google Gemini API key.
7. Ensure the AWS Load Balancer Controller is installed and public ALB subnets are tagged for discovery.
8. The pipeline keeps the existing branch convention: `main` deploys to `pdf-bot-prod`; `Development` deploys to `pdf-bot-dev`.

## Validate
- `helm lint helm/pdf-bot`
- Run Jenkins pipeline and confirm rollout.
- `kubectl get deploy,pods,svc,ingress,hpa,configmap,secret -n pdf-bot-prod`
- `kubectl get ingress pdf-bot -n pdf-bot-prod`
- Test `http://<ALB-DNS>/demo`

## Important application-state limitation
The current app saves FAISS data to local `faiss_index/` and expects later requests to read it. Pod filesystems are ephemeral, and replicas do not share this directory. For this starter deployment, HPA min/max are both 1 to avoid routing a user's question to another pod that lacks the uploaded index. This means the HPA object is present but will not scale out. Before enabling multiple replicas, redesign document/index storage as durable shared state (for example, S3-backed document/index handling with session-aware metadata) and review Streamlit session behavior. EFS alone is not a complete solution for concurrent FAISS index writes.

## Secrets
The pipeline injects the Google API key from Jenkins Credentials using `--set-file`; do not commit API keys to Git or values.yaml. Helm stores release values in Kubernetes, so for production prefer AWS Secrets Manager with External Secrets Operator or Secrets Store CSI Driver and make the chart reference an externally managed Secret.

## Path prefix
Docker starts Streamlit with `--server.baseUrlPath=demo`; the ALB Ingress routes `/demo` and checks `/demo/_stcore/health`. Validate with the installed Streamlit version and inspect pod logs if health checks fail.
