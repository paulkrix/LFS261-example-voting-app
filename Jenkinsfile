pipeline {

  agent none

  stages{
    stage("worker-build"){
      when{
          changeset "**/worker/**"
        }

      agent{
        docker{
          image 'maven:3.6.1-jdk-8-slim'
          args '-v $HOME/.m2:/root/.m2'
        }
      }

      steps{
        echo 'Compiling worker app..'
        dir('worker'){
          sh 'mvn compile'
        }
      }
    }
    stage("worker-test"){
      when{
        changeset "**/worker/**"
      }
      agent{
        docker{
          image 'maven:3.6.1-jdk-8-slim'
          args '-v $HOME/.m2:/root/.m2'
        }
      }
      steps{
        echo 'Running Unit Tets on worker app..'
        dir('worker'){
          sh 'mvn clean test'
          }

        }
    }
    stage("worker-package"){
      when{
        branch 'master'
        changeset "**/worker/**"
      }
      agent{
        docker{
          image 'maven:3.6.1-jdk-8-slim'
          args '-v $HOME/.m2:/root/.m2'
        }
      }
      steps{
        echo 'Packaging worker app'
        dir('worker'){
          sh 'mvn package -DskipTests'
          archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
        }

      }
    }

    stage('worker-docker-package'){
        agent any
        when{
          changeset "**/worker/**"
          branch 'master'
        }
        steps{
          echo 'Packaging worker app with docker'
          script{
            docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
                def workerImage = docker.build("paulkrix/worker:v${env.BUILD_ID}", "./worker")
                workerImage.push()
                workerImage.push("${env.BRANCH_NAME}")
                workerImage.push("latest")
            }JenkinsGithubCreds
          }
        }
    }

    // stage("vote-build"){
    //   when{
    //       changeset "**/vote/**"
    //     }

    //   steps{
    //     echo 'Compiling vote app..'
    //     dir('vote'){
    //       sh "docker build . -t paulkrix/vote:v${env.BUILD_ID}}"
    //     }
    //   }
    // }

    stage('vote-integration') {
      agent any
      when {
        changeset "**/vote/**"
        branch 'master'
      }
      steps {
        echo 'Running Integration Tests on vote app'
        dir('vote') {
          sh 'sh integration_test.sh'
        }
      }
    }

    stage('vote-docker-package'){
        agent any

        when {
            changeset "**/vote/**"
            branch 'master'
        }

        steps {
            echo 'Packaging vote app with docker'
            script{
                docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
                    def voteImage = docker.build("paulkrix/vote:v${env.BUILD_ID}", "./vote")
                    voteImage.push()
                    voteImage.push("${env.BRANCH_NAME}")
                    voteImage.push("latest")
                }
            }
        }
    }

    stage ('result-build') {
        when {
            changeset "**/result/**"
        } 
        
        steps {
            echo 'Compiling result app..' 
            dir ('result') { 
                sh 'npm install'
            }
        }
    }
    
    stage ('result-test') {
        when { 
            changeset "**/result/**"
        } 
        
        steps {
            echo 'Running Unit Tests on result app..'
            dir ('result') {
                sh 'npm install'
                sh 'npm test'
            }
        }
    }
    stage('result-docker-package'){
        agent any
        when{
            changeset "**/result/**"
            branch 'master'
        }
        steps{
            echo 'Packaging vote app with docker'
            script {
                docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
                    def voteImage = docker.build("paulkrix/result:v${env.BUILD_ID}", "./result")
                    voteImage.push()
                    voteImage.push("${env.BRANCH_NAME}")
                    voteImage.push("latest")
                }
            }
        }
    }
    stage ('SonarQube') {
      agent any
      when {
        branch 'master'
      }
      environment {
        sonarpath = tool 'SonarScanner'
      }
      steps {
        echo 'Running SonarQube Analysis...'
        withSonarQubeEnv('sonar-instavote') {
          sh "${sonarpath}/bin/sonar-scanner -Dproject.settings=sonar-project.properties -Dorg.jenkinsci.plugins.durabletask.BourneShellScript.HEARTBEAT_CHECK_INTERVAL=86400"
        }
      }
    }
    stage('Quality Gate') {
      steps {
        timeout(time: 1, unit: 'HOURS') {
          waitForQualityGate abortPipeline: true
        }
      }
    }
    stage('deploy-to-dev') {
      agent any
      when {
        branch 'master'
      }
      steps {
        echo 'Deploy instavote app with docker compose'
        sh 'docker-compose up -d'
      }
    }
    stage('Trigger deployment') {
      agent any
      environment{
        // def GIT_COMMIT = "${env.GIT_COMMIT}"
        def GIT_COMMIT = "${env.BUILD_ID}"
      }
      steps{
        echo "${GIT_COMMIT}"
        echo "triggering deployment"
        // passing variables to job deployment run by vote-deploy repository Jenkinsfile
        build job: 'deployment', parameters: [string(name: 'DOCKERTAG', value: "v${env.BUILD_ID"})]
      }    
   }
  }

  post{
    always{
        echo 'Building multibranch pipeline for worker is completed..'
    }
  }
}