pipeline {
  agent any
  environment {
    DOCKER_HUB_USER = 'vantaicn'
    DOCKER_HUB_PWD = credentials('docker_hub') // Lưu trong Jenkins Credential
    IMAGE_TAG = ''
    // GIT_CREDENTIALS_ID = 'github-token' // Dùng để push lại repo Helm
    // HELM_REPO_DIR = '../spring-petclinic' // Path đến helm chart repo
  }

  stages {
    stage('Prepare') {
      steps {
        script {
          // Lấy commit ID nếu là push hoặc PR
          if (env.GIT_COMMIT) {
            IMAGE_TAG = env.GIT_COMMIT.take(7)
          }

          // Nếu là tag release -> dùng tag làm image tag
          if (env.GIT_TAG_NAME) {
            IMAGE_TAG = env.GIT_TAG_NAME
          }

          echo "Image tag sẽ dùng: ${IMAGE_TAG}"
        }
      }
    }

    stage('Detect Changed Service') {
      steps {
        script {
          def changedFiles = sh(script: "git diff --name-only HEAD~1", returnStdout: true).trim()
          def matchedService = ''

          changedFiles.split('\n').each { file ->
            if (file.contains("spring-petclinic-") && file.contains("-service/")) {
              matchedService = file.split('/')[0]
            }
          }

          if (!matchedService) {
            error("Không phát hiện service nào bị thay đổi.")
          }

          env.TARGET_SERVICE = matchedService
          echo "Service bị thay đổi: ${env.TARGET_SERVICE}"
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        dir("${env.TARGET_SERVICE}") {
          sh "../mvnw clean install -P buildDocker -Dcontainer.image.tag=${IMAGE_TAG}"
        }
      }
    }

    stage('Push Docker Image') {
      steps {
        script {
          def imageName = "${DOCKER_HUB_USER}/${env.TARGET_SERVICE}:${IMAGE_TAG}"
          sh """
            echo "${DOCKER_HUB_PWD}" | docker login -u "${DOCKER_HUB_USER}" --password-stdin
            docker tag ${env.TARGET_SERVICE}:latest ${imageName}
            docker push ${imageName}
          """
        }
      }
    }

    // stage('Update Helm values') {
    //   steps {
    //     script {
    //       def envFile = ''
    //       if (env.GIT_BRANCH == 'main') {
    //         envFile = 'values-dev.yaml'
    //       } else if (env.GIT_TAG_NAME?.startsWith('v')) {
    //         envFile = 'values-staging.yaml'
    //       } else {
    //         error("Không xác định môi trường (dev/staging)")
    //       }

    //       def service = env.TARGET_SERVICE.replace("spring-petclinic-", "").replace("-service", "")
    //       def helmValuesFile = "${HELM_REPO_DIR}/${envFile}"

    //       echo "Cập nhật image cho service ${service} trong file ${helmValuesFile}"

    //       sh """
    //         yq e '.${service}.image.tag = "${IMAGE_TAG}"' -i ${helmValuesFile}
    //       """
    //     }
    //   }
    // }

    // stage('Commit & Push to Git') {
    //   steps {
    //     dir("${HELM_REPO_DIR}") {
    //       withCredentials([usernamePassword(credentialsId: "${GIT_CREDENTIALS_ID}", usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
    //         sh """
    //           git config user.email "ci@example.com"
    //           git config user.name "jenkins"
    //           git add .
    //           git commit -m "Update image for ${env.TARGET_SERVICE} to ${IMAGE_TAG}"
    //           git push https://${GIT_USER}:${GIT_PASS}@your-git-repo-url HEAD:main
    //         """
    //       }
    //     }
    //   }
    // }
  }
}
