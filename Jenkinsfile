def apps = ['web', 'api']

pipeline {
    agent {
        docker {
            image 'node:22-alpine'
            args '-u root' // optional if you need root for installs
        }
    }

    environment {
        DOCKER_BUILDKIT = '1'
        COMPOSE_DOCKER_CLI_BUILD = '1'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Tools') {
            steps {
                sh '''
                    npm install -g turbo
                    apk add --no-cache jq
                '''
            }
        }

        stage('Determine Changed Apps') {
            steps {
                script {
                    def changedApps = []
                    for (app in apps) {
                        def result = sh(
              script: "turbo run build --filter=${app}^... --dry=json | jq '.tasks | length'",
              returnStdout: true
            ).trim()

                        if (result != '0') {
                            changedApps << app
            } else {
                            echo "Skipping ${app} — no changes detected"
                        }
                    }

                    if (changedApps.isEmpty()) {
                        echo 'No apps changed — skipping build'
                        currentBuild.result = 'SUCCESS'
                        skipRemainingStages()
          } else {
                        echo "Changed apps: ${changedApps}"
                    }
                }
            }
        }

        stage('Build and Push Docker Images') {
            matrix {
                axes {
                    axis {
                        name 'APP_NAME'
                        values 'web', 'admin'
                    }
                }
                when {
                    expression { return changedApps.contains(APP_NAME) }
                }
                stages {
                    stage('Build Image') {
                        steps {
                            script {
                                def imageName = "myorg/${APP_NAME}"
                                sh "docker build -f apps/${APP_NAME}/Dockerfile -t ${imageName}:latest ."
                            }
                        }
                    }

                    stage('Push Image') {
                        steps {
                            withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                                sh 'echo "$PASSWORD" | docker login -u "$USERNAME" --password-stdin'
                                sh "docker push myorg/${APP_NAME}:latest"
                            }
                        }
                    }
                }
            }
        }
    }
}
