// Jenkins declarative pipeline: build -> push -> deploy on every push to master.
//
// Required Jenkins credentials (Manage Jenkins -> Credentials):
//   dockerhub-creds  : Username with password (Docker Hub username + access token)
//   deploy-ssh-key   : SSH Username with private key (user on the deploy server, e.g. ubuntu)
//   deploy-host      : Secret text (public IP / DNS of the EC2 VM)
//
// Required plugins: Git, GitHub, Pipeline, SSH Agent, Credentials Binding.

pipeline {
    agent any

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
                withCredentials([string(credentialsId: 'deploy-host', variable: 'DEPLOY_HOST')]) {
                    sshagent(credentials: ['deploy-ssh-key']) {
                        sh '''
                          test -f docker-compose.yml || { echo "ERROR: docker-compose.yml not found"; exit 1; }

                          SSH="ssh -o StrictHostKeyChecking=accept-new $DEPLOY_USER@$DEPLOY_HOST"

                          $SSH "mkdir -p $DEPLOY_DIR"
                          scp -o StrictHostKeyChecking=accept-new docker-compose.yml $DEPLOY_USER@$DEPLOY_HOST:$DEPLOY_DIR/

                          $SSH "cd $DEPLOY_DIR && docker compose pull && docker compose up -d --remove-orphans && docker compose ps"
                          $SSH "docker image prune -f"
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