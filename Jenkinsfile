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
  }
}
