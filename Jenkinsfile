pipeline {
  agent any

  environment {
    TURBO_VERSION = 'latest'
    DOCKER_REGISTRY = 'hub.docker.com' 
  }

  stages {
    stage('Install Tools') {
      agent {
        docker {
          image 'node:22-alpine'
          args '-u root'
        }
      }
      steps {
        sh '''
          npm install -g turbo
          apk add --no-cache jq git docker-cli
        '''
      }
    }

    stage('Detect Changed Apps') {
      agent {
        docker {
          image 'node:22-alpine'
          args '-u root'
        }
      }
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

    stage('Build & Push Matrix') {
      matrix {
        axes {
          axis {
            name 'APP_NAME'
            values 'web', 'api' // include all possible apps
          }
        }
        when {
          expression {
            return env.CHANGED_APPS?.split(',')?.contains(APP_NAME)
          }
        }
        agent {
          docker {
            image 'node:22-alpine'
            args '-u root'
          }
        }
        stages {
          stage('Build Docker Image') {
            steps {
              script {
                def imageTag = "${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER}"
                sh """
                  apk add --no-cache docker-cli
                  docker build -f apps/${APP_NAME}/Dockerfile -t ${imageTag} .
                  echo "Built image: ${imageTag}"
                """
                env.IMAGE_TAG = imageTag
              }
            }
          }

          stage('Push Docker Image') {
            steps {
              withCredentials([usernamePassword(credentialsId: 'docker-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                sh """
                  echo "${DOCKER_PASS}" | docker login ${DOCKER_REGISTRY} -u "${DOCKER_USER}" --password-stdin
                  docker push ${env.IMAGE_TAG}
                """
              }
            }
          }
        }
      }
    }
  }
}
