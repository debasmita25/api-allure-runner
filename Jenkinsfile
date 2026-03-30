pipeline {

    agent any

    parameters {
        choice(name: 'TEST_ENV', choices: ['qa','stage','prod'], description: 'Environment')
        string(name: 'TEST_SUITE', defaultValue: 'testng.xml', description: 'TestNG Suite')
    }

    environment {
        // Optional, can be used in Docker run if needed
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
                        docker rm -f api-tests-runner || true
                        docker compose down || true
                        docker compose up --pull always --abort-on-container-exit
                        EXIT_CODE=$?
                        if [ $EXIT_CODE -ne 0 ]; then exit $EXIT_CODE; fi
                        '''
                    } else {
                        bat """
                        docker rm -f api-tests-runner >nul 2>&1 || exit 0
                        docker compose down
                        docker compose up --pull always --abort-on-container-exit
                        if %ERRORLEVEL% NEQ 0 exit /b %ERRORLEVEL%
                        """
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
                        if [ ! -d "allure-results" ]; then
                            echo "No allure-results folder found!"
                            exit 1
                        fi
                        ${allureHome}/bin/allure generate allure-results --clean -o allure-report
                        zip -r allure-report.zip allure-report
                        """
                    } else {
                        powershell """
                        if (!(Test-Path "allure-results")) {
                            Write-Host "No allure-results folder found!"
                            exit 1
                        }
                        & '${allureHome}\\bin\\allure.bat' generate allure-results --clean -o allure-report
                        Compress-Archive -Path allure-report\\* -DestinationPath allure-report.zip
                        """
                    }
                }
            }
        }

        // stage('Extract Test Summary') {
        //     steps {
        //         script {
        //             if (isUnix()) {
        //                 summary = sh(
        //                     script: "docker logs api-tests-runner | grep -A 2 'GitHub API Test Suite'",
        //                     returnStdout: true
        //                 ).trim()
        //             } else {
        //                 summary = bat(
        //                     script: 'docker logs api-tests-runner | findstr /C:"GitHub API Test Suite"',
        //                     returnStdout: true
        //                 ).trim()
        //             }
        //             env.TEST_SUMMARY = summary
        //         }
        //     }
        // }
        stage('Extract Test Summary') {
        steps {
            script {
                // declare local variable to avoid memory leak warning
                def summary = ""

                if (isUnix()) {
                    // Unix/Linux: extract header + 2 lines after
                    summary = sh(
                        script: "docker logs api-tests-runner 2>/dev/null | grep -A 2 'GitHub API Test Suite' || true",
                        returnStdout: true
                    ).trim()
                } else {
                    // Windows: PowerShell extract header + 2 lines after
                    summary = powershell(
                        script: '''
                        try {
                            $logs = docker logs api-tests-runner 2>$null
                            $lines = $logs -split "`n"
                            $idx = $lines.IndexOf($lines | Where-Object { $_ -match "GitHub API Test Suite" })
                            if ($idx -ge 0) {
                                $lines[$idx..($idx+2)] -join "`n"
                            } else { "" }
                        } catch { "" }
                        ''',
                        returnStdout: true
                    ).trim()
                }

                // assign to env variable to use later in email
                env.TEST_SUMMARY = summary

                echo "✅ Extracted API Test Summary:\n${env.TEST_SUMMARY}"
            }
        }
    }
    }

    post {
        always {
            script {
                archiveArtifacts artifacts: 'allure-results/**', fingerprint: true
                archiveArtifacts artifacts: 'allure-report.zip', fingerprint: true

                emailext(
                    subject: "Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}",
                    to: 'debasmita25@gmail.com',
                    mimeType: 'text/html',
                    attachLog: true,
                    body: """
                    <h3>Build Summary</h3>
                    <p><b>Job:</b> ${env.JOB_NAME}</p>

                    <p><b>Status:</b> 
                    <span style="color: white; background-color: ${currentBuild.currentResult == 'SUCCESS' ? 'green' : 'red'}; padding: 5px;">
                    ${currentBuild.currentResult}
                    </span>
                    </p>

                    <p><a href="${env.BUILD_URL}allure">View Allure Report in Jenkins</a></p>
                    <p><a href="${env.BUILD_URL}artifact/allure-report.zip">Download Allure Report</a></p>

                    <h4>API Test Summary:</h4>
                    <pre>${env.TEST_SUMMARY}</pre>

                    <p style="color:green;">Navigate to the download directory and execute:</p>
                    <p><b>allure open allure-report</b></p>
                    """
                )
            }
        }

        success { echo "✅ Pipeline Passed" }
        failure { echo "❌ Pipeline Failed" }
    }
}