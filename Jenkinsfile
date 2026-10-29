
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


    }

    post {
        success {
            echo 'SUCCESS: BugTracker image built, Django checks and tests passed, and image pushed to Amazon ECR.'
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
