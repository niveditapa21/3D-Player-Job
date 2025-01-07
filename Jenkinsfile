pipeline {
    agent any

    parameters {
        string(name: 'DEPLOYMENT_SERVER', defaultValue: '54.158.53.154', description: 'The IP address of the server to deploy to')
        string(name: 'PORT', defaultValue: '8080', description: 'The port to deploy the application on (between 8000 and 9000)')
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'The Git branch to build from')
    }

    environment {
        JOB_NAME = '3d-player-build-job'
        JENKINS_URL = 'http://54.158.53.154:8080/'
        USERNAME = 'nive'  
        TOKEN = 'd2831a51-6891-4701-8d25-35be2a4af298' 
    }

    stages {
        stage('Trigger Jenkins Parameterized Job') {
            steps {
                script {
                    def crumbResponse = sh(script: """
                        curl -s -u $USERNAME:$TOKEN "${JENKINS_URL}crumbIssuer/api/json"
                    """, returnStdout: true)

                    def crumbJson = readJSON text: crumbResponse
                    def crumb = crumbJson.crumb
                    def crumbRequestField = crumbJson.crumbRequestField

                    def deploymentServerEncoded = URLEncoder.encode(params.DEPLOYMENT_SERVER, 'UTF-8')
                    def portEncoded = URLEncoder.encode(params.PORT, 'UTF-8')
                    def branchNameEncoded = URLEncoder.encode(params.BRANCH_NAME, 'UTF-8')

                    def triggerResponse = sh(script: """
                        curl -v -X POST -u $USERNAME:$TOKEN -H "$crumbRequestField: $crumb" \
                        "${JENKINS_URL}job/$JOB_NAME/buildWithParameters?DEPLOYMENT_SERVER=$deploymentServerEncoded&PORT=$portEncoded&BRANCH_NAME=$branchNameEncoded" -i
                    """, returnStdout: true)

                    echo "Curl Response: $triggerResponse"

                    def queueUrl = triggerResponse.find(/Location: (.*)/) { match -> match[1].trim() }
                    if (!queueUrl) {
                        error 'Failed to trigger Jenkins job. Queue URL not found.'
                    }

                    echo "Triggered Jenkins job. Queue URL: $queueUrl"
                    env.QUEUE_URL = queueUrl
                }
            }
        }

        stage('Wait for Build to Start') {
            steps {
                script {
                    def buildNumber = null
                    for (int i = 0; i < 30; i++) {
                        sleep 5
                        def queueResponse = sh(script: """
                            curl -s -u $USERNAME:$TOKEN "${env.QUEUE_URL}api/json"
                        """, returnStdout: true)

                        def queueJson = readJSON text: queueResponse
                        buildNumber = queueJson.executable?.number
                        if (buildNumber) {
                            break
                        }
                        echo "Waiting for build to start... Attempt: ${i + 1}"
                    }

                    if (!buildNumber) {
                        error 'Failed to retrieve build number after multiple attempts.'
                    }

                    echo "Build number: $buildNumber"
                    env.BUILD_NUMBER = buildNumber
                }
            }
        }

        stage('Check Jenkins Job Status') {
            steps {
                script {
                    def status = null
                    for (int i = 0; i < 60; i++) {
                        sleep 10
                        def buildInfo = sh(script: """
                            curl -s -u $USERNAME:$TOKEN "${JENKINS_URL}job/$JOB_NAME/${env.BUILD_NUMBER}/api/json"
                        """, returnStdout: true)

                        def buildJson = readJSON text: buildInfo
                        status = buildJson.result
                        if (status) {
                            break
                        }
                        echo "Waiting for build to complete... Attempt: ${i + 1}"
                    }

                    if (!status) {
                        error 'Failed to fetch the status of the Jenkins job after multiple attempts.'
                    }

                    echo "Job Status: $status"
                    if (status == 'FAILURE') {
                        def buildLog = sh(script: """
                            curl -s -u $USERNAME:$TOKEN "${JENKINS_URL}job/$JOB_NAME/${env.BUILD_NUMBER}/consoleText"
                        """, returnStdout: true)
                        echo "Build failed. Log:\n${buildLog.take(500)}"
                    }

                    env.BUILD_STATUS = status
                }
            }
        }
    }

    post {
        always {
            script {
                echo "Build Completed. Status: ${env.BUILD_STATUS ?: 'UNKNOWN'}"
                if (env.BUILD_STATUS == 'SUCCESS') {
                    echo "Build was successful."
                } else if (env.BUILD_STATUS == 'FAILURE') {
                    error "Build failed."
                }
            }
        }
    }
}
