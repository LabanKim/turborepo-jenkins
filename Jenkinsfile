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

    stage('Install Dependencies') {
      steps {
        sh 'npm install'
      }
    }

    stage('Detect Changed Apps') {
      steps {
        script {
          // Run turbo to detect changed apps (returns JSON)
          def changedAppsJson = sh(
            script: "turbo run build --dry=json",
            returnStdout: true
          ).trim()

          // Parse changed packages using jq
          def changedAppsStr = sh(
            script: "echo '${changedAppsJson}' | jq -r '[.tasks[].package] | unique | join(\",\")'",
            returnStdout: true
          ).trim()

          env.CHANGED_APPS = changedAppsStr
          echo "Changed apps: ${env.CHANGED_APPS}"
        }
      }
    }

    stage('List All Apps') {
      steps {
        script {
          // List all directories in apps/
          def appsList = sh(
            script: "ls -d apps/* | xargs -n 1 basename",
            returnStdout: true
          ).trim().split("\n")

          env.ALL_APPS = appsList.join(",")
          echo "All apps: ${env.ALL_APPS}"
        }
      }
    }

    stage('Build Changed Apps') {
      steps {
        script {
          def changedApps = (env.CHANGED_APPS?.split(",") ?: []) as List
          def allApps = (env.ALL_APPS?.split(",") ?: []) as List

          def appsToBuild = []
          for (app in allApps) {
            if (changedApps.contains(app)) {
              appsToBuild << app
            }
          }

          if (appsToBuild.isEmpty()) {
            echo "No apps to build"
          } else {
            def buildSteps = [:]
            for (app in appsToBuild) {
              buildSteps[app] = {
                stage("Build ${app}") {
                  sh "turbo run build --filter=${app} --force"
                }
              }
            }
            // parallel buildSteps
          }
        }
      }
    }

    stage('Docker Build & Push') {
      when {
        expression { return env.APPS_TO_DOCKERIZE }
      }
      environment {
        DOCKER_CLI_EXPERIMENTAL = 'enabled'
      }
      steps {
        withCredentials([usernamePassword(credentialsId: '6ef2844b-7697-4d02-aeec-de30293d4e6b', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
          script {
            def apps = env.APPS_TO_DOCKERIZE.split(",")
            sh 'echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin'

            for (app in apps) {
              def tag = "${DOCKER_USERNAME}/${app}:latest"
              sh """
                docker build -t ${tag} apps/${app}
                docker push ${tag}
              """
            }
          }
        }
      }
    }
  }
}
