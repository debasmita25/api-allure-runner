pipeline {
    agent any

    parameters {
        choice(name: 'TEST_ENV', choices: ['qa', 'stage', 'prod'], description: 'Environment')
        string(name: 'TEST_SUITE', defaultValue: 'testng.xml', description: 'TestNG Suite')
    }

    environment {
        TEST_IMAGE = 'debasmita25/github-api-tests:latest'
    }

    stages {

        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Setup Workspace') {
            steps {
                script {
                    if (isUnix()) {
                sh 'rm -rf allure-results allure-report allure-report.zip; mkdir -p allure-results'
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
                        writeFile file: '.env', text: """GITHUB_USERNAME=${GITHUB_USERNAME}
GITHUB_TOKEN=${GITHUB_TOKEN}
TEST_ENV=${TEST_ENV}
TEST_SUITE=${TEST_SUITE}"""

                        if(isUnix()) {
                            sh '''
                                docker pull ''' + env.TEST_IMAGE + '''
                                docker compose down || true
                                docker compose up --abort-on-container-exit
                              '''
                        } else {
                            powershell   """
                                docker pull $env:TEST_IMAGE
                                docker compose down
                                docker compose up --abort-on-container-exit 2>&1 | Tee-Object -FilePath docker-output.txt
                                        if (\$LASTEXITCODE -ne 0) { exit \$LASTEXITCODE }
                                    """
                                }

                                // Extract the test summary line
                                def dockerOutput = readFile('docker-output.txt')
                                def summaryLine = dockerOutput.readLines().find { it.contains('Total tests run:') }
                                env.TEST_SUMMARY = summaryLine ? summaryLine.trim() : 'Test summary not found'
                                
                    }
                }
            }
        }

        stage('Generate Allure Report') {
            steps {
                script {
                    def allureHome = tool 'Allure'
                    isUnix()
                        ? sh("""
                            ${allureHome}/bin/allure generate allure-results --clean -o allure-report
                            zip -r allure-report.zip allure-report
                          """)
                        : powershell("""
                            & '${allureHome}\\bin\\allure.bat' generate allure-results --clean -o allure-report
                            Compress-Archive -Path allure-report\\* -DestinationPath allure-report.zip
                          """)
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'allure-results/**, allure-report.zip', fingerprint: true
            emailext(
                subject: "Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}",
                to: 'debasmita25@gmail.com',
                mimeType: 'text/html',
                body: """
                    <h3>Build Summary</h3>
                    <p><b>Job:</b> ${env.JOB_NAME}</p>
                    <p><b>Status:</b>
                        <span style="color: ${currentBuild.currentResult == 'SUCCESS' ? 'green' : 'red'};">
                            ${currentBuild.currentResult}
                        </span>
                    </p>
                    <p><b>Test Results:</b>
                        <span style="color: green;">${env.TEST_SUMMARY}</span>
                    </p>
                    <p><a href="${env.BUILD_URL}allure">View Allure Report in Jenkins</a></p>
                    <p><a href="${env.BUILD_URL}artifact/allure-report.zip">Download Allure Report</a></p>
                    <p><span style="color: green;">To view locally, run:</span> <b>allure open allure-report</b></p>
                """
            )
        }
        success { echo "✅ Pipeline Passed" }
        failure { echo "❌ Pipeline Failed" }
    }
}