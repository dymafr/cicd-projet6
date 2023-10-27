pipeline {
    agent any
    environment {
        DOCKERHUB_ACCOUNT = 'dymafr'
        FRONT_REPOSITORY = 'cicd-projet6-frontend'
        API_REPOSITORY = 'cicd-projet6-node-api'
        SERVER_USER = 'root'
        SERVER_PRODUCTION = '141.95.149.180'
        SERVER_STAGING = '141.94.245.67'
    }
    stages {
       stage('Build & Push') {
            parallel {
                stage('Frontend') {
                    stages {
                        stage('Checks Frontend') {
                            agent {
                                docker {
                                    image 'node:lts'
                                }
                            }
                            steps {
                                script {
                                    dir('frontend') {
                                        sh 'npm ci'
                                        sh 'npm run lint'
                                        sh 'npm audit'
                                    }
                                }
                            }
                        }
                        stage('Construire image Docker Frontend') {
                            steps {
                                script {
                                    dir('frontend') {
                                        dockerFrontendImage = docker.build("${DOCKERHUB_ACCOUNT}/${FRONT_REPOSITORY}:latest")
                                    }
                                }
                            }
                        }
                        stage('Pousser sur Docker Hub Frontend') {
                            steps {
                                script {
                                    withDockerRegistry([credentialsId: 'docker-hub']) {
                                        dockerFrontendImage.push()
                                    }
                                }
                            }
                        }
                    }
                }
                stage('API') {
                    stages {
                        stage('Checks Node API') {
                            agent {
                                docker {
                                    image 'node:lts-bullseye'
                                }
                            }
                            steps {
                                script {
                                    dir('node-api') {
                                        sh 'npm ci'
                                        sh 'npm run lint'
                                        sh 'npm audit'
                                        sh 'npm run test:ci'
                                    }
                                }
                            }
                        }
                        stage('Construire image Docker Node API') {
                            steps {
                                script {
                                    dir('node-api') {
                                        dockerAPIImage = docker.build("${DOCKERHUB_ACCOUNT}/${API_REPOSITORY}:latest")
                                    }
                                }
                            }
                        }
                        stage('Pousser sur Docker Hub Node API') {
                            steps {
                                script {
                                    withDockerRegistry([credentialsId: 'docker-hub']) {
                                        dockerAPIImage.push()
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Tests E2E') {
            steps {
                script {
                    withDockerRegistry([credentialsId: 'docker-hub']) {
                        sh 'docker compose -f docker-compose.yml up --exit-code-from cypress --abort-on-container-exit'
                    }
                }
                sh 'docker compose -f docker-compose.yml down -v --remove-orphans'
            }
        }
       stage('déployer sur le serveur de staging') {
            environment {
                DOCKER_HUB = credentials('docker-hub')
            }
            steps {
                sshagent(credentials: ['ovh_staging_key']) {
                    sh '''
                        [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                        ssh-keyscan -H $SERVER_STAGING >> ~/.ssh/known_hosts
                        ssh $SERVER_USER@$SERVER_STAGING 'mkdir -p /var/www'
                        scp docker-compose.prod.yml $SERVER_USER@$SERVER_STAGING:/var/www/docker-compose.prod.yml
                        ssh $SERVER_USER@$SERVER_STAGING "docker login -u $DOCKER_HUB_USR -p $DOCKER_HUB_PSW &&
            docker compose -f /var/www/docker-compose.prod.yml up -d --force-recreate"
                    '''
                }
            }
        }
        stage('déployer sur le serveur de production') {
            environment {
                DOCKER_HUB = credentials('docker-hub')
            }
            steps {
                input message: 'Déployer sur le serveur de production ?', ok: 'Oui'
                sshagent(credentials: ['ovh_prod_key']) {
                    sh '''
                        [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                        ssh-keyscan -H $SERVER_PRODUCTION >> ~/.ssh/known_hosts
                        ssh $SERVER_USER@$SERVER_PRODUCTION 'mkdir -p /var/www'
                        scp docker-compose.prod.yml $SERVER_USER@$SERVER_PRODUCTION:/var/www/docker-compose.prod.yml
                        ssh $SERVER_USER@$SERVER_PRODUCTION "docker login -u $DOCKER_HUB_USR -p $DOCKER_HUB_PSW &&
            docker compose -f /var/www/docker-compose.prod.yml up -d --force-recreate"
                    '''
                }
            }
        }
    }
}