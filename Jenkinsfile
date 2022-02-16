pipeline {
  agent {
    label 'docker-agent'
  }
  options {
    ansiColor('xterm')
  }

  environment {
    /*---AWS ECR Credentials---*/
    REGISTRY = '279899492521.dkr.ecr.eu-west-3.amazonaws.com'  // An existing ECR registry name where images will be pushed
    REGISTRY_CREDENTIAL = 'jenkins-AWS-access'                 // Jenkins credentialsID for AWS access
    ECR_REPOSITORY = 'jenkins-demo'                            // ECR repository name
    ECR_REGION = 'eu-west-3'                                   // AWS region

    /*----SSH Server details---*/
    SERVER_NAME = 'ubuntu@13.38.126.29'                        // Remote server name where artifacts will be deployed
    REMOTE_DIRECTORY = 'artifacts'                             // Creates, if given directory does not exists

    /*---Docker Build configuration---*/
    VERSION = 'latest'
    DOCKERFILE_PATH = '/home/ubuntu/artifacts/'

    /*---Github Config---*/
    BRANCH_NAME = 'main'
    SCM_CREDENTIALS = 'github'                                 // Credentials ID stored in Jenkins
    REPOSITORY_URL = 'git@github.com:andrasKelle/cc-jenkins-demo.git'
  }

  stages {
    stage('Source'){
      steps {
        git branch: "${BRANCH_NAME}",
            credentialsId: "${SCM_CREDENTIALS}",
            url: "${SCM_URL}"
      }
    }

    stage('Build') {
      steps {
        sh """
          docker info
          docker build -t ${ECR_REPOSITORY}:${BUILD_NUMBER} .
          docker tag ${ECR_REPOSITORY}:${BUILD_NUMBER} ${REGISTRY}/${ECR_REPOSITORY}:${VERSION}
          docker images
        """
      }
    }

    stage('Deploy') {
      steps {
        sh "echo \033[33m Deploy ${ECR_REPOSITORY}:${BUILD_NUMBER} to AWS ECR \033[0m"
        script {
          docker.withRegistry("https://${REGISTRY}", "ecr:${ECR_REGION}:${REGISTRY_CREDENTIAL}") {
              docker.image("${REGISTRY}/${ECR_REPOSITORY}").push("${VERSION}")
          }
        }
      }
    }

    stage('Post-Deploy') {
      steps {
        sh 'echo \033[33m Creating artifacts... \033[0m'
        archiveArtifacts artifacts: '**/*', fingerprint: true
        
        sh 'echo \033[33m Publish artifacts to remote server \033[0m'
        sshPublisher(publishers: [sshPublisherDesc(configName: "${SERVER_NAME}", transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: "docker build -t ${ECR_REPOSITORY}:${VERSION} ${DOCKERFILE_PATH} && docker run -dp 80:80 ${ECR_REPOSITORY}:${VERSION}", execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: "${REMOTE_DIRECTORY}", remoteDirectorySDF: false, removePrefix: '', sourceFiles: '**/*')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
      }
    }
  }

  post {
    success {
      sh "echo \033[32m Successfully builded docker image and pushed it to ECR. ${ECR_REPOSITORY}:${BUILD_NUMBER} now running on ${SERVER_NAME} \033[0m" 
    }
    failure {
      sh 'echo \033[31m Failed \033[0m'
    }
  }
}