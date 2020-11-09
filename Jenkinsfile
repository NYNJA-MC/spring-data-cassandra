#!/usr/bin/env groovy

@Library('nynja-common') _

pipeline {
    environment {
        SLACK_CHANNEL = "#nynja-devops-feed"
        NAMESPACE = "nynja-spring-data-cassandra"
        APP_NAME = "nynja-spring-data"
        IMAGE_NAME = "eu.gcr.io/nynja-ci-201610/${NAMESPACE}/${APP_NAME}"
        IMAGE_BUILD_TAG = "$BRANCH_NAME-$BUILD_NUMBER"
        HELM_CHART_NAME = "nynja-spring-data"
        DEV_BRANCH = "dev"
    }
    agent {
        kubernetes(builders.multi([
                "mvn":"maven:3-jdk-10",
                "helm":"lachlanevenson/k8s-helm:v2.9.1"
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
                    slackSend channel: SLACK_CHANNEL, message: slackStartMsg()
                    slackSend channel: SLACK_CHANNEL, message: "", attachments: slackBuildInfo()
                }
            }
        }
        stage('Build PR') {
            when {
                branch 'PR-*'
            }
            stages {
                stage('Build') {
                    steps {
                        container('mvn') {
                            withCredentials([file(credentialsId: 'mavenSettings.xml', variable: 'FILE')]) {
                                sh 'mvn -U --settings $FILE clean install -DskipTests=true'
                            }
                        }
                    }
                }
            }
        }
        stage('Build commits') {
            when {
                not {
                    anyOf {
                        branch env.DEV_BRANCH;
                        branch 'master'
                        branch 'PR-*'
                    }
                }
            }
            stages {
                stage('Build') {
                    steps {
                        container('mvn') {
                            withCredentials([file(credentialsId: 'mavenSettings.xml', variable: 'FILE')]) {
                                sh 'mvn -U --settings $FILE clean install -DskipTests=true'
                            }
                        }
                    }
                }
            }
        }

        stage('Build Dev') {
            when {
                branch env.DEV_BRANCH
            }
            stages {
                stage('Build') {
                    steps {
                        container('mvn') {
                            withCredentials([file(credentialsId: 'mavenSettings.xml', variable: 'FILE')]) {
                                sh 'mvn -U --settings $FILE clean install -DskipTests=true'
                            }
                            dockerBuildAndPushToRegistry "${NAMESPACE}/${APP_NAME}", [IMAGE_BUILD_TAG]
                        }
                    }
                }
                stage("Helm chart") {
                    steps {
                        container('helm') {
                            helmBuildAndPushToRegistry HELM_CHART_NAME
                        }
                    }
                }
                stage('Deploy preview') {
                    steps {
                        deployHelmTo "dev", NAMESPACE
                    }
                }
            }
            post {
                success {
                    container('mvn') { slackSend channel: SLACK_CHANNEL, message: slackEndMsg(), color: 'good' }
                }
                failure {
                    container('mvn') { slackSend channel: SLACK_CHANNEL, message: slackEndMsg(), color: 'danger' }
                }
            }
        }
        stage('Build Release') {
            when {
                branch 'master'
            }
            stages {
                stage("Build") {
                    steps {
                        container('mvn') {
                            withCredentials([file(credentialsId: 'mavenSettings.xml', variable: 'FILE')]) {
                                sh 'mvn -U --settings $FILE clean install -DskipTests=true'
                            }
                            dockerBuildAndPushToRegistry "${NAMESPACE}/${APP_NAME}", [IMAGE_BUILD_TAG]
                        }
                    }
                }
                stage("Helm chart") {
                    steps {
                        container('helm') {
                            helmBuildAndPushToRegistry HELM_CHART_NAME
                        }
                    }
                }
                stage("Approval: Deploy to staging ?") {
                    steps {
                        slackSend channel: SLACK_CHANNEL, message: "$APP_NAME: build #$BUILD_NUMBER ready to deploy to `STAGING`, approval required: $BUILD_URL (24h)"

                        timeout(time: 24, unit: 'HOURS') { input 'Deploy to staging ?' }
                    }
                    post { failure { echo 'Deploy aborted for build #...' }}
                }
                stage("Deploy to staging") {
                    steps {
                        slackSend channel: SLACK_CHANNEL, message: "$APP_NAME: deploying build #$BUILD_NUMBER to `STAGING`"
                        deployHelmTo "staging", NAMESPACE
                    }
                }
                stage("Approval: Deploy to production ?") {
                    steps {
                        slackSend channel: SLACK_CHANNEL, message: "$APP_NAME: build #$BUILD_NUMBER ready to deploy to `PRODUCTION`, approval required: $BUILD_URL (24h)"

                        timeout(time: 7, unit: 'DAYS') { input 'Deploy to production ?' }
                    }
                    post { failure { echo 'Deploy aborted for build #...' }}
                }
                stage('Tagging release') {
                    steps {
                        container("mvn") {
                            // Updating the "latest tag"
                            dockerTagLatestAndPushToRegistry "${NAMESPACE}/${APP_NAME}", IMAGE_BUILD_TAG
                        }
                    }
                }
                /*
                stage('Deploy release to canary') {
                  steps {
                    slackSend channel: SLACK_CHANNEL, message: "$APP_NAME: deploying build #$BUILD_NUMBER to `PRODUCTION` (canary)"
                    echo "deploy to canary"
                  }
                }
                */
                stage("Deploy to production") {
                    steps {
                        slackSend channel: SLACK_CHANNEL, message: "$APP_NAME: deploying build #$BUILD_NUMBER to `PRODUCTION`"

                        deployHelmTo "prod", NAMESPACE
                    }
                }
            }
        }
    }
}
