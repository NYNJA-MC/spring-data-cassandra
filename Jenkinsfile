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
		    withCredentials([usernamePassword(credentialsId: 'artifactory-global-publisher', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
			sh '''echo > settings.xml '<?xml version="1.0" encoding="UTF-8"?><settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd"> <servers> <server> <id>nynjagroup.jfrog.io</id> <username>'${USER}'</username> <password>'${PASS}'</password> </server> <server> <id>nynjagroup.jfrog.io-libs-release</id> <username>'${USER}'</username> <password>'${PASS}'</password> </server> </servers> <profiles> <profile> <id>nynja</id> <repositories> <repository> <id>nynjagroup.jfrog.io</id> <url>https://nynjagroup.jfrog.io/nynjagroup</url> </repository> <repository> <id>nynjagroup.jfrog.io-libs-release</id> <name>libs-release</name> <url>https://nynjagroup.jfrog.io/nynjagroup/libs-release</url> </repository> </repositories> </profile> </profiles> <activeProfiles> <activeProfile>nynja</activeProfile> </activeProfiles> </settings>'
'''
			sh 'mvn --settings settings.xml clean install deploy -DskipTests=true -DaltSnapshotDeploymentRepository=nynjagroup.jfrog.io::default::https://nynjagroup.jfrog.io/nynjagroup/libs-snapshot-local'
		    }

 	  // withCredentials([usernamePassword(credentialsId: 'helm-publisher', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
	  // 	sh """
          //          echo "machine nynjagroup.jfrog.io" > ~/.netrc;
          //          echo "login $USER" >> ~/.netrc;
          //          echo "password $PASS" >> ~/.netrc;
          //          echo curl -n -T ./target/${name} "https://nynjagroup.jfrog.io/nynjagroup/libs-release-local/${name}" 
          //       """
          // }
          // }
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
