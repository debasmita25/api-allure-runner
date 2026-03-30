pipeline {

    agent any

    parameters {
        choice(name: 'TEST_ENV', choices: ['qa','stage','prod'], description: 'Select the Test Environment')
        string(name: 'TEST_SUITE', defaultValue: 'testng.xml', description: 'Enter the TestNG suite file')
    }

    environment {
        TEST_IMAGE = 'debasmita25/github-api-tests:latest'
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Pull Test Image') {
            steps {
                script {
                    if (isUnix()) {
                        sh "docker pull ${TEST_IMAGE}"
                    } else {
                        bat "docker pull %TEST_IMAGE%"
                    }
                }
            }
        }

        stage('Pre-Clean Setup') {
            steps {
                script {
                    if (isUnix()) {
                        sh '''
                        rm -rf allure-results allure-report allure-report.zip || true
                        mkdir -p allure-results allure-report
                        '''
                    } else {
                        bat '''
                        IF EXIST allure-results rmdir /s /q allure-results
                        IF EXIST allure-report rmdir /s /q allure-report
                        IF EXIST allure-report.zip del /f /q allure-report.zip
                        mkdir allure-results
                        mkdir allure-report
                        '''
                    }
                }
            }
        }

        stage('Run API Tests') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'api-creds',
                        usernameVariable: 'GITHUB_USERNAME',
                        passwordVariable: 'GITHUB_TOKEN'
                    )]) {

                        if (isUnix()) {

                            sh '''
                            set -e

                            cat <<EOF > .env
GITHUB_USERNAME=$GITHUB_USERNAME
GITHUB_TOKEN=$GITHUB_TOKEN
TEST_ENV=$TEST_ENV
TEST_SUITE=$TEST_SUITE
EOF

                            docker compose down || true
                            docker compose up --abort-on-container-exit
                            '''

                        } else {

                            powershell '''
                            $ErrorActionPreference = "Stop"

                            @"
GITHUB_USERNAME=$env:GITHUB_USERNAME
GITHUB_TOKEN=$env:GITHUB_TOKEN
TEST_ENV=$env:TEST_ENV
TEST_SUITE=$env:TEST_SUITE
"@ | Out-File -Encoding ASCII .env

                            try {
                                docker compose down
                            } catch {
                                Write-Host "Ignore cleanup error"
                            }

                            docker compose up --abort-on-container-exit
                            '''
                        }
                    }
                }
            }
        }

        stage('Post Cleanup (Containers)') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'docker compose down || true'
                    } else {
                        powershell '''
                        try {
                            docker compose down
                        } catch {
                            Write-Host "Ignore cleanup error"
                        }
                        '''
                    }
                }
            }
        }

        stage('Check Allure Results') {
            steps {
                script {
                    if (isUnix()) {
                        sh '''
                        if [ -d "allure-results" ] && [ "$(ls -A allure-results)" ]; then
                            echo "Allure results present"
                        else
                            echo "Allure results missing or empty"
                            exit 1
                        fi
                        '''
                    } else {
                         powershell '''
                            if ((Test-Path "allure-results") -and ((Get-ChildItem "allure-results").Count -gt 0)) {
                                Write-Host "Allure results found"
                            } else {
                                Write-Host "No Allure results found"
                                exit 1
                            }
                            '''
                    }
                }
            }
        }

        stage('Generate Allure HTML (CLI)') {
            steps {
                script {
                    def allureHome = tool 'Allure'

                    if (isUnix()) {
                        sh "${allureHome}/bin/allure generate allure-results --clean -o allure-report"
                    } else {
                        powershell "& '${allureHome}\\bin\\allure.bat' generate allure-results --clean -o allure-report"
                    }
                }
            }
        }

        stage('Zip Allure Report') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'zip -r allure-report.zip allure-report'
                    } else {
                        powershell 'Compress-Archive -Path allure-report\\* -DestinationPath allure-report.zip'
                    }
                }
            }
        }
    }

        stage('Validate Report Zip') {
        steps {
            script {
                if (isUnix()) {
                    sh 'ls -l allure-report.zip || exit 1'
                } else {
                    powershell 'if (!(Test-Path "allure-report.zip")) { exit 1 }'
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'allure-results/**', fingerprint: true
            archiveArtifacts artifacts: 'allure-report.zip', fingerprint: true

          emailext(
            subject: "Build ${env.BUILD_NUMBER} - ${currentBuild.currentResult}",
            body: """
                <h2>Jenkins Build Report</h2>

                <p><b>Job:</b> ${env.JOB_NAME}</p>
                <p><b>Build:</b> ${env.BUILD_NUMBER}</p>
                <p><b>Status:</b> ${currentBuild.currentResult}</p>

                <p><b>Allure Report:</b></p>
                <p><a href="${env.BUILD_URL}allure">Click here to view report</a></p>

                <br/>
                <p><b>Attached:</b> Allure HTML Report (ZIP)</p>
                <p><b>Run command to view:</b> allure open allure-report </p>

                <br/>
                <p>Regards,<br/>Jenkins</p>
            """,
            mimeType: 'text/html',
            to: 'debasmita25@gmail.com',
            attachmentsPattern: 'allure-report.zip',
            compressLog: true,
            attachLog: true   // ✅ useful for debugging
        )
        }

        success {
            echo "✅ Pipeline Passed"
        }

        failure {
            echo "❌ Pipeline Failed"
        }
    }
}