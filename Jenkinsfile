pipeline {
  agent any
  environment {
    DOCKERHUB_LOGIN = credentials('jenkins-docker-login')
    DOCKERHUB_PASSWORD = credentials('jenkins-docker-password')
  }
  triggers {
    pollSCM('H/1 * * * *')
  }
  options {
    buildDiscarder(logRotator(numToKeepStr: '5')) 
  }
  stages {
    stage('Build') {
      agent {
        docker { 
          image 'maven:3.5-jdk-8' 
          args '-v /maven/.m2:/root/.m2'
        }
      }
      steps {
        sh "mvn clean install"
        stash name: "warfile", includes: "petclinic/target/petclinic.war"
      }
      post {
        always {
          junit '**/target/surefire-reports/**/*.xml'
        }
      }
    }
    stage('BuildDocker') {
      steps {
        unstash name: "warfile"
        sh "docker build -t $DOCKERHUB_LOGIN/petclinic:$BUILD_NUMBER ."
        sh "docker push $DOCKERHUB_LOGIN/petclinic:$BUILD_NUMBER"
      }
    }
    stage('IntegrationTesting') {
      parallel {
        stage('EndToEnd') {
          steps {
            script{
              docker.image("$DOCKERHUB_LOGIN/petclinic:$BUILD_NUMBER").withRun('-p 48080:8080') { container ->
                docker.image("maven:3.5-jdk-8").inside("--link=${container.id}:localhost"){
                  sh "mvn verify -Pselenium-tests -Dselenium.port=48080 -pl petclinic_it"
                }
              }
            }
          }
        }
        stage('LastTest') {
          steps {
              sh "docker run -d --name dockerLT -p 58080:8080 $DOCKERHUB_LOGIN/petclinic:$BUILD_NUMBER"
              sh "mvn verify -Pjmeter-tests -pl petclinic_it"
          }
          post {
            always {
              junit '**/target/surefire-reports/**/*.xml'
              sh "docker stop dockerLT"
              sh "docker rm dockerLT"
            }
          }
        }
      }
    }
    stage('Deploy') {
      steps {
        sh "echo deploy"
      }
    } 
  }
}
