#!/usr/bin/env groovy

@Library('nynja-common') _

pipeline {
  environment {
	SLACK_CHANNEL = "#nynja-devops-feed"
	LIB_NAME="nynja-spring-data-cassandra"
  }
  agent {
    kubernetes(builders.multi([
      "mvn":"maven:3-jdk-10",
    ]))

  }
  options {
    skipDefaultCheckout()
    buildDiscarder(logRotator(numToKeepStr: '15'))
  }
  stages {
    stage('Checkout') {
      steps {
        container('mvn') {
          script {
            def vars = checkout scm
            vars.each { k,v -> env.setProperty(k, v) }
          }
        }
      }
    }
    stage('Build') {
      when { not { changeRequest() } }
      steps {
        container('mvn') {
          withCredentials([file(credentialsId: 'mavenSettings.xml', variable: 'FILE')]) {
			sh 'mvn --settings $FILE clean install -DskipTests=true'
			sh 'ls -l'
          }
        }
      }
    }
    stage('Publish') {
      when { anyOf{
		branch 'development'
		branch 'master'
	    } }
      steps {
        container('jdk') {
          withCredentials([usernamePassword(credentialsId: 'artifactory-global-publisher', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
            // sh 'echo "" >> gradle.properties'
            // sh 'echo "nynjagroup_jfrog_io_user = $USER" >> gradle.properties'
            // sh 'echo "nynjagroup_jfrog_io_password = $PASS" >> gradle.properties'
            // sh './gradlew --no-daemon publish'
          }
        }
      }
    }
  }
}
