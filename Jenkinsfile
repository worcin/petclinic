pipeline {
  agent any
  environment {
    DOCKERHUB_LOGIN = credentials('jenkins-docker-login')
    DOCKERHUB_PASSWORD = credentials('jenkins-docker-password')
  }
  triggers {
    upstream(upstreamProjects: 'Upstream', threshold: hudson.model.Result.SUCCESS)
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
        sh "mvn -B clean install"
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
        sh 'docker login -u $DOCKERHUB_LOGIN -p $DOCKERHUB_PASSWORD'
        sh 'docker build -t $DOCKERHUB_LOGIN/petclinic:$BUILD_NUMBER .'
        sh 'docker push $DOCKERHUB_LOGIN/petclinic:$BUILD_NUMBER'
      }
    }
    stage('IntegrationTesting') {
      parallel {
        stage('EndToEnd') {
          steps {
            script{
              docker.image("$DOCKERHUB_LOGIN/petclinic:$BUILD_NUMBER").withRun { container ->
                docker.image("maven:3.5-jdk-8").inside("--link=${container.id}:selenium -P -v /maven/.m2:/root/.m2"){
                  sh 'echo test'
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
              archiveArtifacts '**/target/jmeter/testFiles/**/*.jmx'
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
