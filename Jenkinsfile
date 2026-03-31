pipeline {

    agent any

    parameters {
        choice(name: 'TEST_ENV', choices: ['qa','stage','prod'], description: 'Environment')
        string(name: 'TEST_SUITE', defaultValue: 'testng.xml', description: 'TestNG Suite')
    }

    environment {
        TEST_IMAGE = 'debasmita25/github-api-tests:latest'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Setup Workspace') {
            steps {
                script {
                    if (isUnix()) {
                        sh '''
                            rm -rf allure-results allure-report allure-report.zip || true
                            mkdir -p allure-results
                        '''
                    } else {
                        bat '''
                            IF EXIST allure-results rmdir /s /q allure-results
                            IF EXIST allure-report rmdir /s /q allure-report
                            IF EXIST allure-report.zip del /f /q allure-report.zip
                            mkdir allure-results
                        '''
                    }
                }
            }
        }

        stage('Run Tests (Docker)') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'api-creds',
                        usernameVariable: 'GITHUB_USERNAME',
                        passwordVariable: 'GITHUB_TOKEN'
                    )]) {
                        if (isUnix()) {
                            sh '''
                                cat <<EOF > .env
GITHUB_USERNAME=$GITHUB_USERNAME
GITHUB_TOKEN=$GITHUB_TOKEN
TEST_ENV=$TEST_ENV
TEST_SUITE=$TEST_SUITE
EOF
                                docker compose down || true
                                docker compose up --pull always --abort-on-container-exit
                            '''
                        } else {
                            powershell '''
@"
GITHUB_USERNAME=$env:GITHUB_USERNAME
GITHUB_TOKEN=$env:GITHUB_TOKEN
TEST_ENV=$env:TEST_ENV
TEST_SUITE=$env:TEST_SUITE
"@ | Out-File -Encoding ASCII .env

# Stop and remove existing containers safely
docker compose down -v -t 10 2>$null

# Run docker compose with always pull latest image
docker compose up --pull always --abort-on-container-exit
'''
                        }
                    }
                }
            }
        }

        stage('Generate Allure Report') {
            steps {
                script {
                    def allureHome = tool 'Allure'
                    if (isUnix()) {
                        sh """
                            ${allureHome}/bin/allure generate allure-results --clean -o allure-report
                            zip -r allure-report.zip allure-report
                        """
                    } else {
                        powershell """
& '${allureHome}\\bin\\allure.bat' generate allure-results --clean -o allure-report
Compress-Archive -Path allure-report\\* -DestinationPath allure-report.zip
"""
                    }
                }
            }
        }

        stage('Extract Test Summary') {
            steps {
                script {
                    def summary = ""
                    if (isUnix()) {
                        summary = sh(script: "docker logs api-tests-runner | grep 'GitHub API Test Suite' -A 3", returnStdout: true).trim()
                    } else {
                        summary = powershell(script: "docker logs api-tests-runner | Select-String 'GitHub API Test Suite' -Context 0,3", returnStdout: true).trim()
                    }
                    env.API_TEST_SUMMARY = summary
                }
            }
        }

    } // stages

    post {
        always {
            archiveArtifacts artifacts: 'allure-results/**', fingerprint: true
            archiveArtifacts artifacts: 'allure-report.zip', fingerprint: true

            script {
                def emailBody = """
                    <h3>Build Summary</h3>
                    <p><b>Job:</b> ${env.JOB_NAME}</p>
                    <p><b>Status:</b> 
                        <span style="color: ${currentBuild.currentResult == 'SUCCESS' ? 'green' : 'red'};">
                        ${currentBuild.currentResult}
                        </span>
                    </p>

                    <h4>API Test Summary:</h4>
                    <pre>${env.API_TEST_SUMMARY}</pre>

                    <p><a href="${env.BUILD_URL}allure">View Allure Report in Jenkins</a></p>
                    <p><a href="${env.BUILD_URL}artifact/allure-report.zip">Download Allure Report</a></p>

                    <p><span style="color:green;">Navigate to the download directory and execute:</span></p>
                    <p><b>allure open allure-report</b></p>
                """

                emailext(
                    subject: "Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}",
                    to: 'debasmita25@gmail.com',
                    mimeType: 'text/html',
                    body: emailBody,
                    attachmentsPattern: "${env.WORKSPACE}/allure-report.zip"
                )
            }
        }

        success { echo "✅ Pipeline Passed" }
        failure { echo "❌ Pipeline Failed" }
    }
}