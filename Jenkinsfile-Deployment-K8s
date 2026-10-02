pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        AWS_REGION         = 'ap-south-1'
        AWS_ACCOUNT_ID     = '992839646359'

        DEPLOY_ENV         = ''

        EKS_CLUSTER        = ''
        ECR_REPOSITORY     = ''
        ECR_REGISTRY       = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_REPOSITORY_URL = ''

        POD_IAM_ROLE_NAME  = ''
        RDS_IDENTIFIER     = ''
        JWT_SECRET_ID      = ''
        ALB_SECURITY_GROUP_NAME = ''

        NAMESPACE           = 'shopsphere'

        // NEW: SonarQube project
        SONAR_PROJECT_KEY  = 'shopsphere-application'
        SONAR_PROJECT_NAME = 'ShopSphere Application'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm

                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()

                    echo "Building Git Commit: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Determine Environment') {
            steps {
                script {

                    if (env.BRANCH_NAME == 'develop') {
                        env.DEPLOY_ENV = 'dev'
                    }
                    else if (env.BRANCH_NAME == 'qa') {
                        env.DEPLOY_ENV = 'qa'
                    }
                    else if (env.BRANCH_NAME == 'main') {
                        env.DEPLOY_ENV = 'prod'
                    }
                    else {
                        error(
                            "Unsupported branch: ${env.BRANCH_NAME}. " +
                            "Use develop, qa, or main."
                        )
                    }

                    env.EKS_CLUSTER =
                        "shopsphere-${env.DEPLOY_ENV}-eks"

                    env.ECR_REPOSITORY =
                        "${env.DEPLOY_ENV}-shopsphere"

                    env.ECR_REPOSITORY_URL =
                        "${env.ECR_REGISTRY}/${env.ECR_REPOSITORY}"

                    env.POD_IAM_ROLE_NAME =
                        "${env.DEPLOY_ENV}-shopsphere-pod"

                    env.RDS_IDENTIFIER =
                        "shopsphere-${env.DEPLOY_ENV}-postgres"

                    env.JWT_SECRET_ID =
                        "${env.DEPLOY_ENV}/shopsphere/jwt-secret"

                    env.ALB_SECURITY_GROUP_NAME =
                        "shopsphere-${env.DEPLOY_ENV}-alb"

                    echo "========================================"
                    echo " ENVIRONMENT CONFIGURATION"
                    echo "========================================"
                    echo "Git Branch             : ${env.BRANCH_NAME}"
                    echo "Environment            : ${env.DEPLOY_ENV}"
                    echo "EKS Cluster            : ${env.EKS_CLUSTER}"
                    echo "ECR Repository         : ${env.ECR_REPOSITORY}"
                    echo "RDS Identifier         : ${env.RDS_IDENTIFIER}"
                    echo "JWT Secret ID          : ${env.JWT_SECRET_ID}"
                    echo "ALB Security Group     : ${env.ALB_SECURITY_GROUP_NAME}"
                    echo "========================================"
                }
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
                    set -e

                    echo "==== Java ===="
                    java -version

                    echo "==== Maven ===="
                    mvn -version

                    echo "==== Node ===="
                    node --version

                    echo "==== NPM ===="
                    npm --version

                    echo "==== Docker ===="
                    docker --version

                    echo "==== AWS CLI ===="
                    aws --version

                    echo "==== kubectl ===="
                    kubectl version --client

                    echo "==== Helm ===="
                    helm version --short

                    # NEW: SonarQube Scanner
                    echo "==== SonarQube Scanner ===="
                    sonar-scanner --version

                    # NEW: Trivy
                    echo "==== Trivy ===="
                    trivy --version
                '''
            }
        }

        stage('AWS Authentication') {
            steps {
                sh '''
                    set -e

                    echo "==== AWS Identity ===="
                    aws sts get-caller-identity

                    echo "==== AWS Region ===="
                    echo "$AWS_REGION"
                '''
            }
        }

        stage('Backend Tests') {
            steps {
                sh '''
                    set -e

                    echo "Testing product-service"

                    cd application-scr/services/product-service
                    mvn -B clean test
                    cd ../../..

                    echo "Testing user-service"

                    cd application-scr/services/user-service
                    mvn -B clean test
                    cd ../../..

                    echo "Testing order-service"

                    cd application-scr/services/order-service
                    mvn -B clean test
                    cd ../../..

                    echo "Testing payment-service"

                    cd application-scr/services/payment-service
                    mvn -B clean test
                    cd ../../..
                '''
            }
        }

        stage('Frontend Test') {
            steps {
                sh '''
                    set -e

                    cd application-scr/frontend

                    npm ci
                    npm run build
                '''
            }
        }

        // ============================================================
        // NEW: SONARQUBE CODE QUALITY / SECURITY ANALYSIS
        // ============================================================

        stage('SonarQube Code Scan') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        set -e

                        echo "========================================"
                        echo " Starting SonarQube Analysis"
                        echo "========================================"

                        sonar-scanner \
                          -Dsonar.projectKey="$SONAR_PROJECT_KEY" \
                          -Dsonar.projectName="$SONAR_PROJECT_NAME" \
                          -Dsonar.sources=application-scr \
                          -Dsonar.exclusions="**/node_modules/**,**/target/**,**/.env*,**/*.lock" \
                          -Dsonar.java.binaries="application-scr/services/product-service/target/classes,application-scr/services/user-service/target/classes,application-scr/services/order-service/target/classes,application-scr/services/payment-service/target/classes" \
                          -Dsonar.sourceEncoding=UTF-8

                        echo "========================================"
                        echo " SonarQube Analysis Completed"
                        echo "========================================"
                    '''
                }
            }
        }

        // ============================================================
        // NEW: SONARQUBE QUALITY GATE
        // ============================================================

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qualityGate = waitForQualityGate(
                            abortPipeline: true
                        )

                        echo "SonarQube Quality Gate Status: ${qualityGate.status}"

                        if (qualityGate.status != 'OK') {
                            error(
                                "SonarQube Quality Gate failed: " +
                                "${qualityGate.status}"
                            )
                        }
                    }
                }
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    set -e

                    aws ecr get-login-password \
                        --region "$AWS_REGION" | \
                    docker login \
                        --username AWS \
                        --password-stdin "$ECR_REGISTRY"
                '''
            }
        }

        stage('Build Backend Images') {
            steps {
                sh '''
                    set -e

                    echo "Building product-service"

                    docker build \
                        -t "$ECR_REPOSITORY_URL:product-service-$IMAGE_TAG" \
                        application-scr/services/product-service


                    echo "Building user-service"

                    docker build \
                        -t "$ECR_REPOSITORY_URL:user-service-$IMAGE_TAG" \
                        application-scr/services/user-service


                    echo "Building order-service"

                    docker build \
                        -t "$ECR_REPOSITORY_URL:order-service-$IMAGE_TAG" \
                        application-scr/services/order-service


                    echo "Building payment-service"

                    docker build \
                        -t "$ECR_REPOSITORY_URL:payment-service-$IMAGE_TAG" \
                        application-scr/services/payment-service
                '''
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh '''
                    set -e

                    cd application-scr/frontend

                    cat > .env.production <<EOF
VITE_PRODUCT_API=/product
VITE_USER_API=/user
VITE_ORDER_API=/order
VITE_PAYMENT_API=/payment
EOF

                    npm ci
                    npm run build

                    cd ../..

                    docker build \
                        -t "$ECR_REPOSITORY_URL:frontend-$IMAGE_TAG" \
                        application-scr/frontend
                '''
            }
        }

        // ============================================================
        // NEW: TRIVY CONTAINER IMAGE SECURITY SCAN
        // ============================================================

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    set -e

                    echo "========================================"
                    echo " Starting Trivy Image Security Scan"
                    echo "========================================"

                    echo "Scanning product-service..."
                    trivy image \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        "$ECR_REPOSITORY_URL:product-service-$IMAGE_TAG"


                    echo "Scanning user-service..."
                    trivy image \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        "$ECR_REPOSITORY_URL:user-service-$IMAGE_TAG"


                    echo "Scanning order-service..."
                    trivy image \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        "$ECR_REPOSITORY_URL:order-service-$IMAGE_TAG"


                    echo "Scanning payment-service..."
                    trivy image \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        "$ECR_REPOSITORY_URL:payment-service-$IMAGE_TAG"


                    echo "Scanning frontend..."
                    trivy image \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        "$ECR_REPOSITORY_URL:frontend-$IMAGE_TAG"


                    echo "========================================"
                    echo " Trivy Scan Completed Successfully"
                    echo " No HIGH/CRITICAL vulnerabilities found"
                    echo "========================================"
                '''
            }
        }

        stage('Push Images') {
            steps {
                sh '''
                    set -e

                    echo "Pushing product-service"

                    docker push \
                        "$ECR_REPOSITORY_URL:product-service-$IMAGE_TAG"


                    echo "Pushing user-service"

                    docker push \
                        "$ECR_REPOSITORY_URL:user-service-$IMAGE_TAG"


                    echo "Pushing order-service"

                    docker push \
                        "$ECR_REPOSITORY_URL:order-service-$IMAGE_TAG"


                    echo "Pushing payment-service"

                    docker push \
                        "$ECR_REPOSITORY_URL:payment-service-$IMAGE_TAG"


                    echo "Pushing frontend"

                    docker push \
                        "$ECR_REPOSITORY_URL:frontend-$IMAGE_TAG"
                '''
            }
        }

        stage('Configure EKS Access') {
            steps {
                sh '''
                    set -e

                    aws eks update-kubeconfig \
                        --region "$AWS_REGION" \
                        --name "$EKS_CLUSTER"

                    echo "==== EKS Cluster ===="

                    kubectl cluster-info

                    echo "==== Nodes ===="

                    kubectl get nodes
                '''
            }
        }

        stage('Install AWS Load Balancer Controller') {
            steps {
                sh '''
                    set -e

                    helm repo add eks \
                        https://aws.github.io/eks-charts

                    helm repo update

                    helm upgrade --install \
                        aws-load-balancer-controller \
                        eks/aws-load-balancer-controller \
                        --namespace kube-system \
                        --set clusterName="$EKS_CLUSTER" \
                        --set serviceAccount.create=false \
                        --set serviceAccount.name=aws-load-balancer-controller
                '''
            }
        }

        stage('Install Secrets Store CSI Driver') {
            steps {
                sh '''
                    set -e

                    helm repo add secrets-store-csi-driver \
                        https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts

                    helm repo update

                    helm upgrade --install \
                        csi-secrets-store \
                        secrets-store-csi-driver/secrets-store-csi-driver \
                        --namespace kube-system \
                        --set syncSecret.enabled=true \
                        --set enableSecretRotation=true


                    helm repo add aws-secrets-manager \
                        https://aws.github.io/secrets-store-csi-driver-provider-aws

                    helm repo update

                    helm upgrade --install \
                        secrets-provider-aws \
                        aws-secrets-manager/secrets-store-csi-driver-provider-aws \
                        --namespace kube-system
                '''
            }
        }

        stage('Deploy Namespace') {
            steps {
                sh '''
                    set -e

                    kubectl apply \
                        -f kubernetes/namespace/namespace.yaml
                '''
            }
        }

        stage('Deploy ConfigMap') {
            steps {
                sh '''
                    set -e

                    kubectl apply \
                        -f kubernetes/config/configmap.yaml
                '''
            }
        }

        stage('Deploy Service Account') {
            steps {
                sh '''
                    set -e

                    SHOPSPHERE_POD_ROLE_ARN=$(aws iam get-role \
                        --role-name "$POD_IAM_ROLE_NAME" \
                        --query 'Role.Arn' \
                        --output text)

                    test -n "$SHOPSPHERE_POD_ROLE_ARN"
                    test "$SHOPSPHERE_POD_ROLE_ARN" != "None"

                    sed \
                        "s|\\${SHOPSPHERE_POD_ROLE_ARN}|$SHOPSPHERE_POD_ROLE_ARN|g" \
                        kubernetes/service-account/service-account.yaml \
                        > /tmp/shopsphere-service-account.yaml

                    kubectl apply \
                        -f /tmp/shopsphere-service-account.yaml
                '''
            }
        }

        stage('Deploy SecretProviderClass') {
            steps {
                sh '''
                    set -e

                    echo "Getting RDS master secret ARN..."

                    RDS_MASTER_USER_SECRET_ARN=$(aws rds describe-db-instances \
                        --region "$AWS_REGION" \
                        --db-instance-identifier "$RDS_IDENTIFIER" \
                        --query 'DBInstances[0].MasterUserSecret.SecretArn' \
                        --output text)

                    echo "Getting JWT secret ARN..."

                    JWT_SECRET_ARN=$(aws secretsmanager describe-secret \
                        --region "$AWS_REGION" \
                        --secret-id "$JWT_SECRET_ID" \
                        --query 'ARN' \
                        --output text)

                    test -n "$RDS_MASTER_USER_SECRET_ARN"
                    test "$RDS_MASTER_USER_SECRET_ARN" != "None"

                    test -n "$JWT_SECRET_ARN"
                    test "$JWT_SECRET_ARN" != "None"

                    sed \
                        -e "s|<RDS_MASTER_USER_SECRET_ARN>|$RDS_MASTER_USER_SECRET_ARN|g" \
                        -e "s|\\${JWT_SECRET_ARN}|$JWT_SECRET_ARN|g" \
                        kubernetes/config/secret-provider-class.yaml \
                        > /tmp/shopsphere-secret-provider-class.yaml

                    kubectl apply \
                        -f /tmp/shopsphere-secret-provider-class.yaml
                '''
            }
        }

        stage('Get ALB Security Group') {
            steps {
                sh '''
                    set -e

                    ALB_SECURITY_GROUP_ID=$(aws ec2 describe-security-groups \
                        --region "$AWS_REGION" \
                        --filters \
                        "Name=group-name,Values=$ALB_SECURITY_GROUP_NAME" \
                        --query 'SecurityGroups[0].GroupId' \
                        --output text)

                    test -n "$ALB_SECURITY_GROUP_ID"
                    test "$ALB_SECURITY_GROUP_ID" != "None"

                    echo "ALB Security Group found"

                    sed \
                        "s|sg-xxxxxxxx|$ALB_SECURITY_GROUP_ID|g" \
                        kubernetes/ingress/ingress.yaml \
                        > /tmp/shopsphere-ingress.yaml
                '''
            }
        }

        stage('Production Approval') {
            when {
                branch 'main'
            }
            steps {
                input(
                    message: 'Application deployment to PROD is ready. Do you want to continue?',
                    ok: 'Deploy Application to PROD'
                )
            }
        }

        stage('Deploy Frontend') {
            steps {
                sh '''
                    set -e

                    kubectl apply \
                        -f kubernetes/frontend/deployment.yaml

                    kubectl apply \
                        -f kubernetes/frontend/service.yaml

                    kubectl -n "$NAMESPACE" set image \
                        deployment/frontend \
                        frontend="$ECR_REPOSITORY_URL:frontend-$IMAGE_TAG"
                '''
            }
        }

        stage('Deploy Product Service') {
            steps {
                sh '''
                    set -e

                    kubectl apply \
                        -f kubernetes/product-service/deployment.yaml

                    kubectl apply \
                        -f kubernetes/product-service/service.yaml

                    kubectl -n "$NAMESPACE" set image \
                        deployment/product-service \
                        product-service="$ECR_REPOSITORY_URL:product-service-$IMAGE_TAG"
                '''
            }
        }

        stage('Deploy User Service') {
            steps {
                sh '''
                    set -e

                    kubectl apply \
                        -f kubernetes/user-service/deployment.yaml

                    kubectl apply \
                        -f kubernetes/user-service/service.yaml

                    kubectl -n "$NAMESPACE" set image \
                        deployment/user-service \
                        user-service="$ECR_REPOSITORY_URL:user-service-$IMAGE_TAG"
                '''
            }
        }

        stage('Deploy Order Service') {
            steps {
                sh '''
                    set -e

                    kubectl apply \
                        -f kubernetes/order-service/deployment.yaml

                    kubectl apply \
                        -f kubernetes/order-service/service.yaml

                    kubectl -n "$NAMESPACE" set image \
                        deployment/order-service \
                        order-service="$ECR_REPOSITORY_URL:order-service-$IMAGE_TAG"
                '''
            }
        }

        stage('Deploy Payment Service') {
            steps {
                sh '''
                    set -e

                    kubectl apply \
                        -f kubernetes/payment-service/deployment.yaml

                    kubectl apply \
                        -f kubernetes/payment-service/service.yaml

                    kubectl -n "$NAMESPACE" set image \
                        deployment/payment-service \
                        payment-service="$ECR_REPOSITORY_URL:payment-service-$IMAGE_TAG"
                '''
            }
        }

        stage('Deploy Ingress') {
            steps {
                sh '''
                    set -e

                    kubectl apply \
                        -f /tmp/shopsphere-ingress.yaml
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    set -e

                    echo "=== FRONTEND ==="

                    kubectl -n "$NAMESPACE" rollout status \
                        deployment/frontend \
                        --timeout=5m


                    echo "=== PRODUCT ==="

                    kubectl -n "$NAMESPACE" rollout status \
                        deployment/product-service \
                        --timeout=5m


                    echo "=== USER ==="

                    kubectl -n "$NAMESPACE" rollout status \
                        deployment/user-service \
                        --timeout=5m


                    echo "=== ORDER ==="

                    kubectl -n "$NAMESPACE" rollout status \
                        deployment/order-service \
                        --timeout=5m


                    echo "=== PAYMENT ==="

                    kubectl -n "$NAMESPACE" rollout status \
                        deployment/payment-service \
                        --timeout=5m
                '''
            }
        }

        stage('Deployment Summary') {
            steps {
                sh '''
                    set -e

                    echo ""
                    echo "========================================"
                    echo " SHOPSPHERE AWS DEPLOYMENT"
                    echo "========================================"
                    echo ""

                    echo "Environment:"
                    echo "$DEPLOY_ENV"

                    echo ""
                    echo "==== PODS ===="

                    kubectl -n "$NAMESPACE" get pods -o wide

                    echo ""
                    echo "==== SERVICES ===="

                    kubectl -n "$NAMESPACE" get services

                    echo ""
                    echo "==== INGRESS ===="

                    kubectl -n "$NAMESPACE" get ingress

                    echo ""
                    echo "==== DEPLOYMENTS ===="

                    kubectl -n "$NAMESPACE" get deployments

                    echo ""
                    echo "Image tag:"
                    echo "$IMAGE_TAG"

                    echo ""
                    echo "ECR repository:"
                    echo "$ECR_REPOSITORY_URL"

                    echo ""
                    echo "EKS cluster:"
                    echo "$EKS_CLUSTER"
                '''
            }
        }
    }

    post {

        success {
            echo """
            ======================================
            SHOPSPHERE AWS DEPLOYMENT SUCCESSFUL
            ======================================

            Environment: ${env.DEPLOY_ENV}
            Image tag: ${env.IMAGE_TAG}
            EKS Cluster: ${env.EKS_CLUSTER}
            Namespace: ${env.NAMESPACE}
            ECR Repository: ${env.ECR_REPOSITORY_URL}

            Security:
            SonarQube Quality Gate: PASSED
            Trivy Image Scan: PASSED
            """
        }

        failure {
            echo """
            =================================
            SHOPSPHERE AWS DEPLOYMENT FAILED
            =================================

            Environment: ${env.DEPLOY_ENV}

            Check the failed Jenkins stage.

            Possible security/quality failure:

            - SonarQube Quality Gate
            - Trivy HIGH/CRITICAL vulnerability scan

            Useful commands:

            kubectl -n shopsphere get pods

            kubectl -n shopsphere get events --sort-by=.lastTimestamp

            kubectl -n shopsphere describe pods
            """
        }

        always {
            sh '''
                echo "====== FINAL POD STATUS ======="

                kubectl -n shopsphere get pods 2>/dev/null || true

                echo "============= FINAL INGRESS STATUS ========="

                kubectl -n shopsphere get ingress 2>/dev/null || true
            '''
        }
    }
}


// pipeline ends