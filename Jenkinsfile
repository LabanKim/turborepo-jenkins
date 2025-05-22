pipeline {
    agent {
        docker {
            image 'node:22-alpine'
            args '-u root'
        }
    }

    environment {
        TURBO_VERSION = 'latest'
    }

    stages {
        stage('Install Tools') {
            steps {
                sh '''
                    apk add --no-cache jq git
                    npm install -g turbo
                '''
            }
        }

        stage('Detect Changed Apps') {
            steps {
                script {
                    def output = sh(
                        script: "turbo run build --dry=json | jq -r '[.tasks[].package] | unique | join(\",\")'",
                        returnStdout: true
                    ).trim()
                    env.CHANGED_APPS = output
                    echo "Changed apps: ${env.CHANGED_APPS}"
                }
            }
        }

        stage('Build Changed Apps') {
            matrix {
                axes {
                    axis {
                        name 'APP_NAME'
                        values 'web', 'api' // add all your known app names here
                    }
                }
                when {
                    expression {
                        return env.CHANGED_APPS?.split(',')?.contains(APP_NAME)
                    }
                }
                steps {
                    sh "turbo run build --filter=${APP_NAME}"
                }
            }
        }
    }
}
