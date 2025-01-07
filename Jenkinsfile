pipeline {
    agent any

    parameters {
        string(name: 'DEPLOYMENT_SERVER', defaultValue: '45.79.122.237', description: 'The IP address of the server to deploy to')
        string(name: 'PORT', defaultValue: '8000', description: 'The port to deploy the application on (between 8000 and 9000)')
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'The Git branch to build from')
    }

    environment {
        JOB_NAME = '3d-player-build-job'
        JENKINS_URL = 'http://54.158.53.154:8080'
        USERNAME = 'niveditapa21'
        TOKEN = ${{ secrets.TOKEN }}
    }

    stages {
        stage('Trigger Jenkins Parameterized Job') {
            steps {
                script {
                    def deploymentServerEncoded = URLEncoder.encode(params.DEPLOYMENT_SERVER, 'UTF-8')
                    def portEncoded = URLEncoder.encode(params.PORT, 'UTF-8')
                    def branchNameEncoded = URLEncoder.encode(params.BRANCH_NAME, 'UTF-8')

                    
                    def triggerResponse = sh(script: """
                        curl -X POST -u $USERNAME:$TOKEN "$JENKINS_URL/job/$JOB_NAME/buildWithParameters?DEPLOYMENT_SERVER=$deploymentServerEncoded&PORT=$portEncoded&BRANCH_NAME=$branchNameEncoded" -i
                    """, returnStdout: true)

                    
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

                        buildNumber = queueResponse.tokenize('\n').find { it.contains('"executable":') }?.tokenize(':')?.last()?.trim()
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
                            curl -s -u $USERNAME:$TOKEN "$JENKINS_URL/job/$JOB_NAME/$BUILD_NUMBER/api/json"
                        """, returnStdout: true)

                        status = buildInfo.tokenize('\n').find { it.contains('"result":') }?.tokenize(':')?.last()?.trim().replace('"', '')
                        if (status && status != 'null') {
                            break
                        }
                        echo "Waiting for build to complete... Attempt: ${i + 1}"
                    }

                    if (!status || status == 'null') {
                        error 'Failed to fetch the status of the Jenkins job after multiple attempts.'
                    }

                    echo "Job Status: $status"
                    if (status == 'FAILURE') {
                        def buildLog = sh(script: """
                            curl -s -u $USERNAME:$TOKEN "$JENKINS_URL/job/$JOB_NAME/$BUILD_NUMBER/consoleText"
                        """, returnStdout: true)
                        echo "Build failed. Log:\n${buildLog.take(500)}" // Print the first 500 characters of the log
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
