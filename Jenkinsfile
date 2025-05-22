pipeline {
  agent {
    docker {
      image 'node:22-alpine'
      args '-u root'
    }
  }

  environment {
    TURBO_VERSION = 'latest'
    DOCKER_REGISTRY = 'hub.docker.com'
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

    stage('Detect Changed Apps') {
        steps {
            script {
            // Run turbo to detect changed packages (apps + packages)
            def changedAppsJson = sh(
                script: "turbo run build --dry=json",
                returnStdout: true
            ).trim()

            // Extract only unique package names from turbo output
            def changedPackages = sh(
                script: "echo '${changedAppsJson}' | jq -r '[.tasks[].package] | unique | join(\",\")'",
                returnStdout: true
            ).trim().split(",")

            // Filter changedPackages against ALL_APPS
            def knownApps = env.ALL_APPS?.split(",") ?: []
            def changedApps = changedPackages.findAll { knownApps.contains(it) }

            env.CHANGED_APPS = changedApps.join(",")
            echo "Changed apps: ${env.CHANGED_APPS}"
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
          }
        }
      }
    }

    stage('Docker Build') {
      when {
        expression { return env.CHANGED_APPS }
      }
      
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
            script {
            def apps = env.CHANGED_APPS.split(",")
            def tags = []

            for (app in apps) {
                def tag = "${DOCKER_USERNAME}/${app}:latest"
                tags << tag

                sh """
                    apk add --no-cache docker-cli
                    docker build -f apps/${app}/Dockerfile -t ${tag} .
                    echo "Built image: ${tag}"
                """
            }

            env.BUILT_DOCKER_TAGS = tags.join(",")
            }
        }
      }
    }

   stage('Docker Push') {
      when {
        expression { return env.BUILT_DOCKER_TAGS }
      }
      environment {
        DOCKER_CLI_EXPERIMENTAL = 'enabled'
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
          script {
            sh 'echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin'

            def tags = env.BUILT_DOCKER_TAGS.split(",")
            for (tag in tags) {
              sh "docker push ${tag}"
            }
          }
        }
      }
    }
  }
}
