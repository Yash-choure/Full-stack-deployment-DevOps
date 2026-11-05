pipeline {
    agent any

    triggers {
        githubPush()
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '15'))
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        DOCKERHUB_USER = 'bunnyop'
        BACKEND_IMAGE  = "${DOCKERHUB_USER}/bunnyidv:bakend"
        FRONTEND_IMAGE = "${DOCKERHUB_USER}/bunnyidv:frontend"
        DEPLOY_USER    = 'ubuntu'
        DEPLOY_DIR     = '/opt/hello-app'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.IMAGE_TAG = sh(script: 'git rev-parse --short=8 HEAD', returnStdout: true).trim()
                }
                echo "Building commit ${env.IMAGE_TAG} on ${env.GIT_BRANCH}"
            }
        }

        stage('Verify (backend + frontend)') {
            parallel {
                stage('Backend checks') {
                    steps {
                        // Runs inside a throwaway container so Jenkins needs no Python install
                        sh '''
                          docker run --rm -v "$PWD/backend":/app -w /app python:3.12-slim sh -c \
                            "pip install --no-cache-dir -r requirements.txt && python manage.py check && python manage.py test"
                        '''
                    }
                }
                stage('Frontend build') {
                    steps {
                        sh '''
                          docker run --rm -v "$PWD/frontend":/app -w /app node:20-alpine sh -c \
                            "npm ci && npm run build"
                        '''
                    }
                }
            }
        }

        stage('Build images') {
            steps {
                sh '''
                  docker build -t $BACKEND_IMAGE ./backend
                  docker build -t $FRONTEND_IMAGE ./frontend
                '''
            }
        }

        stage('Push images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    sh '''
                      echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                      docker push $BACKEND_IMAGE
                      docker push $FRONTEND_IMAGE
                      docker logout
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([string(credentialsId: 'deploy-host', variable: 'DEPLOY_HOST'),
                                 string(credentialsId: 'django-secret-key', variable: 'DJANGO_SECRET_KEY')]) {
                    sshagent(credentials: ['deploy-ssh-key']) {
                        sh '''
                          # Fail fast with a clear message if a file is missing from the repo
                          test -f docker-compose.yml  || { echo "ERROR: docker-compose.yml not found in repo root"; exit 1; }
                          test -f deploy/deploy.sh    || { echo "ERROR: deploy/deploy.sh not found in repo"; exit 1; }

                          SSH="ssh -o StrictHostKeyChecking=accept-new $DEPLOY_USER@$DEPLOY_HOST"

                          $SSH "mkdir -p $DEPLOY_DIR"
                          scp -o StrictHostKeyChecking=accept-new \
                              docker-compose.yml deploy/deploy.sh \
                              $DEPLOY_USER@$DEPLOY_HOST:$DEPLOY_DIR/

                          # Keep a .prod.yml copy on the server so deploy.sh works with either filename
                          $SSH "cp $DEPLOY_DIR/docker-compose.yml $DEPLOY_DIR/docker-compose.prod.yml"

                          # Secrets are passed over the SSH session env, never written to the repo
                          $SSH "cd $DEPLOY_DIR && chmod +x deploy.sh && \
                                BACKEND_IMAGE=$BACKEND_IMAGE FRONTEND_IMAGE=$FRONTEND_IMAGE \
                                IMAGE_TAG=$IMAGE_TAG DJANGO_SECRET_KEY='$DJANGO_SECRET_KEY' \
                                DJANGO_ALLOWED_HOSTS='$DEPLOY_HOST' \
                                ./deploy.sh"
                        '''
                    }
                }
            }
        }
    }

    post {
        success { echo "Deployed ${env.IMAGE_TAG} successfully." }
        failure { echo 'Pipeline failed - check the stage logs above.' }
        always  { sh 'docker image prune -f || true' }
    }
}