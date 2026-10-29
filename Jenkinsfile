
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME = 'bugtracker-ci'
    }

    stages {
        stage('Verify DevOps Repository') {
            steps {
                sh '''
                    set -eu
                    echo "Checking DevOps repository files..."
                    test -f Jenkinsfile
                    test -f docker/Dockerfile
                    test -f docker/Dockerfile.dockerignore
                    echo "DevOps repository validation passed."
                    git rev-parse --short HEAD
                '''
            }
        }

        stage('Checkout Application') {
            steps {
                dir('application') {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: '*/main']],
                        userRemoteConfigs: [[
                            url: 'https://github.com/suryaprakash-r/bugtracker.git',
                            credentialsId: 'github-readonly'
                        ]]
                    ])
                }
            }
        }

        stage('Validate Application Source') {
            steps {
                sh '''
                    set -eu
                    test -f application/manage.py
                    test -f application/requirements.txt
                    test -f application/config/settings.py
                    echo "Application source validation passed."
                    git -C application rev-parse --short HEAD
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu
                    docker build \
                        --file docker/Dockerfile \
                        --tag bugtracker-ci:${BUILD_NUMBER} \
                        application
                '''
            }
        }

        stage('Django System Check') {
            steps {
                sh '''
                    set -eu
                    docker run --rm \
                        -e SECRET_KEY=ci-only-placeholder-not-for-production \
                        -e DEBUG=False \
                        -e ALLOWED_HOSTS=localhost,127.0.0.1 \
                        -e DB_ENGINE=sqlite \
                        -e DB_NAME=/tmp/ci.sqlite3 \
                        bugtracker-ci:${BUILD_NUMBER} \
                        python manage.py check
                '''
            }
        }

        stage('Automated Tests') {
            steps {
                sh '''
                    set -eu

                    docker run --rm \
                        -e SECRET_KEY=ci-only-placeholder-not-for-production \
                        -e DEBUG=False \
                        -e ALLOWED_HOSTS=localhost,127.0.0.1 \
                        -e DB_ENGINE=sqlite \
                        -e DB_NAME=/tmp/ci.sqlite3 \
                        bugtracker-ci:${BUILD_NUMBER} \
                        python manage.py test -v 2
                '''
            }
        }

        stage('Push Image to Amazon ECR') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-ecr-publisher',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''#!/usr/bin/env bash
                        set +x
                        set -euo pipefail

                        ECR_REGISTRY="626209129961.dkr.ecr.ap-south-1.amazonaws.com"
                        ECR_IMAGE="${ECR_REGISTRY}/bugtracker:build-${BUILD_NUMBER}"

                        # Clean up the login and the extra image tag,
                        # even if publishing fails.
                        trap '
                            docker logout "$ECR_REGISTRY" >/dev/null 2>&1 || true
                            docker image rm "$ECR_IMAGE" >/dev/null 2>&1 || true
                        ' EXIT

                        echo "Authenticating to Amazon ECR..."

                        docker run --rm \
                            -e AWS_ACCESS_KEY_ID \
                            -e AWS_SECRET_ACCESS_KEY \
                            -e AWS_DEFAULT_REGION=ap-south-1 \
                            public.ecr.aws/aws-cli/aws-cli:2.39.0 \
                            ecr get-login-password --region ap-south-1 \
                        | docker login \
                            --username AWS \
                            --password-stdin "$ECR_REGISTRY"

                        echo "Tagging image for Amazon ECR..."
                        docker tag \
                            "bugtracker-ci:${BUILD_NUMBER}" \
                            "$ECR_IMAGE"

                        echo "Pushing image to Amazon ECR..."
                        docker push "$ECR_IMAGE"

                        echo "Successfully pushed: $ECR_IMAGE"
                    '''
                }
            }
        }

        stage('Deploy to Amazon EKS') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-eks-deployer',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''#!/usr/bin/env bash
                        set +x
                        set -euo pipefail
                        umask 077

                        AWS_REGION="ap-south-1"
                        CLUSTER_NAME="bugtracker-eks"
                        NAMESPACE="webapps"
                        DEPLOYMENT_NAME="bugtracker"
                        CONTAINER_NAME="bugtracker"
                        ECR_IMAGE="626209129961.dkr.ecr.ap-south-1.amazonaws.com/bugtracker:build-${BUILD_NUMBER}"
                        AWS_CLI_IMAGE="public.ecr.aws/aws-cli/aws-cli:2.39.0"
                        KUBECTL_VERSION="v1.37.1"

                        TMP_DIR="$(mktemp -d)"
                        trap 'rm -rf "$TMP_DIR"' EXIT

                        case "$(uname -m)" in
                            x86_64|amd64) KUBECTL_ARCH="amd64" ;;
                            aarch64|arm64) KUBECTL_ARCH="arm64" ;;
                            *)
                                echo "Unsupported agent architecture." >&2
                                exit 1
                                ;;
                        esac

                        KUBECTL_URL="https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/${KUBECTL_ARCH}/kubectl"

                        # Persistent cache: survives normal Jenkins build cleanup.
                        KUBECTL_CACHE_DIR="${HOME}/.cache/bugtracker-ci/kubectl/${KUBECTL_VERSION}/${KUBECTL_ARCH}"
                        CACHED_KUBECTL="${KUBECTL_CACHE_DIR}/kubectl"
                        KUBECTL=""

                        # 1. Reuse the system-installed binary if the version matches.
                        if command -v kubectl >/dev/null 2>&1; then
                            SYSTEM_KUBECTL="$(command -v kubectl)"
                            SYSTEM_VERSION="$("$SYSTEM_KUBECTL" version \
                                --client --output=yaml 2>/dev/null \
                                | awk '/gitVersion:/ {print $2; exit}' || true)"

                            if [ "$SYSTEM_VERSION" = "$KUBECTL_VERSION" ]; then
                                KUBECTL="$SYSTEM_KUBECTL"
                                echo "Using system kubectl: $SYSTEM_VERSION"
                            fi
                        fi

                        # 2. Otherwise, reuse the persistent cached binary.
                        if [ -z "$KUBECTL" ] && [ -x "$CACHED_KUBECTL" ]; then
                            CACHED_VERSION="$("$CACHED_KUBECTL" version \
                                --client --output=yaml 2>/dev/null \
                                | awk '/gitVersion:/ {print $2; exit}' || true)"

                            if [ "$CACHED_VERSION" = "$KUBECTL_VERSION" ]; then
                                KUBECTL="$CACHED_KUBECTL"
                                echo "Using cached kubectl: $CACHED_VERSION"
                            else
                                echo "Cached kubectl is missing or has an unexpected version."
                                rm -f "$CACHED_KUBECTL"
                            fi
                        fi

                        # 3. Download only when neither existing binary is suitable.
                        if [ -z "$KUBECTL" ]; then
                            echo "No matching kubectl found. Downloading $KUBECTL_VERSION..."
                            mkdir -p "$KUBECTL_CACHE_DIR"

                            curl --fail --location --show-error --progress-bar \
                                --connect-timeout 15 \
                                --max-time 120 \
                                --retry 1 \
                                --retry-delay 3 \
                                --output "$TMP_DIR/kubectl" \
                                "$KUBECTL_URL"

                            curl --fail --silent --show-error --location \
                                --connect-timeout 15 \
                                --max-time 30 \
                                --retry 1 \
                                --retry-delay 3 \
                                --output "$TMP_DIR/kubectl.sha256" \
                                "${KUBECTL_URL}.sha256"

                            echo "Verifying downloaded kubectl checksum..."
                            printf '%s  %s\n' \
                                "$(tr -d '[:space:]' < "$TMP_DIR/kubectl.sha256")" \
                                "$TMP_DIR/kubectl" \
                                | sha256sum --check --status

                            # Cache the verified binary outside TMP_DIR.
                            cp "$TMP_DIR/kubectl" "$CACHED_KUBECTL"
                            chmod 0755 "$CACHED_KUBECTL"
                            KUBECTL="$CACHED_KUBECTL"

                            echo "kubectl verified and cached."
                        fi

                        echo "Selected kubectl client:"
                        "$KUBECTL" version --client

                        echo "Retrieving the EKS cluster endpoint..."
                        EKS_ENDPOINT="$(docker run --rm \
                            -e AWS_ACCESS_KEY_ID \
                            -e AWS_SECRET_ACCESS_KEY \
                            -e AWS_DEFAULT_REGION="$AWS_REGION" \
                            "$AWS_CLI_IMAGE" \
                            eks describe-cluster \
                            --name "$CLUSTER_NAME" \
                            --region "$AWS_REGION" \
                            --query 'cluster.endpoint' \
                            --output text)"

                        EKS_CA_DATA="$(docker run --rm \
                            -e AWS_ACCESS_KEY_ID \
                            -e AWS_SECRET_ACCESS_KEY \
                            -e AWS_DEFAULT_REGION="$AWS_REGION" \
                            "$AWS_CLI_IMAGE" \
                            eks describe-cluster \
                            --name "$CLUSTER_NAME" \
                            --region "$AWS_REGION" \
                            --query 'cluster.certificateAuthority.data' \
                            --output text)"

                        echo "Generating temporary EKS authentication token..."
                        EKS_TOKEN="$(docker run --rm \
                            -e AWS_ACCESS_KEY_ID \
                            -e AWS_SECRET_ACCESS_KEY \
                            -e AWS_DEFAULT_REGION="$AWS_REGION" \
                            "$AWS_CLI_IMAGE" \
                            eks get-token \
                            --cluster-name "$CLUSTER_NAME" \
                            --region "$AWS_REGION" \
                            --query 'status.token' \
                            --output text)"

                        if [ -z "$EKS_ENDPOINT" ] || \
                           [ -z "$EKS_CA_DATA" ] || \
                           [ -z "$EKS_TOKEN" ]; then
                            echo "Failed to obtain EKS connection details." >&2
                            exit 1
                        fi

                        cat > "$TMP_DIR/kubeconfig" <<EOF
apiVersion: v1
kind: Config
clusters:
- name: ${CLUSTER_NAME}
  cluster:
    server: ${EKS_ENDPOINT}
    certificate-authority-data: ${EKS_CA_DATA}
contexts:
- name: ${CLUSTER_NAME}
  context:
    cluster: ${CLUSTER_NAME}
    namespace: ${NAMESPACE}
    user: jenkins-eks-deployer
current-context: ${CLUSTER_NAME}
users:
- name: jenkins-eks-deployer
  user:
    token: ${EKS_TOKEN}
EOF

                        chmod 0600 "$TMP_DIR/kubeconfig"
                        export KUBECONFIG="$TMP_DIR/kubeconfig"

                        echo "Checking access to the existing deployment..."
                        "$KUBECTL" get deployment "$DEPLOYMENT_NAME" \
                            --namespace "$NAMESPACE"

                        echo "Deploying image: $ECR_IMAGE"
                        "$KUBECTL" set image \
                            "deployment/${DEPLOYMENT_NAME}" \
                            "${CONTAINER_NAME}=${ECR_IMAGE}" \
                            --namespace "$NAMESPACE"

                        echo "Waiting for the Kubernetes rollout..."
                        "$KUBECTL" rollout status \
                            "deployment/${DEPLOYMENT_NAME}" \
                            --namespace "$NAMESPACE" \
                            --timeout=180s

                        echo "Verifying the final deployment..."
                        "$KUBECTL" get deployment "$DEPLOYMENT_NAME" \
                            --namespace "$NAMESPACE" \
                            --output wide

                        "$KUBECTL" get pods \
                            --namespace "$NAMESPACE" \
                            --selector app=bugtracker

                        echo "EKS deployment completed successfully."
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: BugTracker built, tested, pushed to Amazon ECR, and deployed successfully to Amazon EKS.'
        }

        failure {
            echo 'FAILURE: Check the stage logs to diagnose the pipeline.'
        }

        always {
            sh '''
                docker image rm \
                    bugtracker-ci:${BUILD_NUMBER} \
                    >/dev/null 2>&1 || true
            '''
        }
    }
}
