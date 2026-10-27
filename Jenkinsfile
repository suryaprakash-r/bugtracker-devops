
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

    }

    post {
        success {
            echo 'SUCCESS: BugTracker image built, Django checks and tests passed.'
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
