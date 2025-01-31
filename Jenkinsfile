pipeline {
  agent any 
  tools {
    nodejs 'nodejs-20.8'
  }
  environment {
    DOCKER_REPO='nguyenmanhtrinh/demo-app'
  }

  stages {
    stage('Increment Version') {
      steps {
        script {
          echo 'update version...'
          sh ''' 
            cd ./app
            npm install
            npx auto-version --patch
          '''

          echo 'Read version'
          def packageJson = readJSON file: './app/package.json'
          def app_version = packageJson.version
          env.IMAGE_NAME = "node-app-${app_version}"
          echo "${env.IMAGE_NAME}"
        }
      }
    }
    stage('Testing stage'){
      steps {
        script {
          echo 'Testing ...'
          sh '''
            cd ./app
            npm run test
          '''
        }
      }
    }
    stage('Build and Push docker images .....'){
      steps {
        script {
          echo 'Building Docker IMAGES'
          withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'USER', passwordVariable: 'PWD')]){
            sh "docker build -t ${env.DOCKER_REPO}:${env.IMAGE_NAME} ."
            sh "echo ${PWD} | docker login -u ${USER} --password-stdin"
            sh "docker push ${env.DOCKER_REPO}:${env.IMAGE_NAME}"
          }
        }
      }
    }
    stage('Deploy To EC2'){
      steps {
        script {
          echo 'Deploying to EC2'
          sshagent(['ec2-credentials']) {
            sh "ssh -o StrictHostKeyChecking=no ec2-user@13.57.187.125 docker run -d -p 3000:3000 ${env.DOCKER_REPO}:${env.IMAGE_NAME}"
          }
        }
      }
    }
    stage ('Automatic Commit to Git repo'){
      steps {
        script {
          echo 'Commit and push to Git Repo .....'
          withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'USER', passwordVariable: 'PWD')]){
            sh "git remote set-url origin https://${USER}:${PWD}@github.com/ManhTrinhNguyen/AWS-exercise"

            sh 'git config --global user.email jenkins@gmail.com'
            sh 'git config --global user.name Jenkins'

            sh 'git status'
            sh 'git config --list'

            sh 'git add .'
            sh 'git commit -m "ci: bump version"'
            sh 'git push origin HEAD:main'
          }
          
        }
      }
    }
  }
}
