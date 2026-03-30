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
                            docker compose up --abort-on-container-exit
                            '''
                        } else {
                            powershell '''
@"
GITHUB_USERNAME=$env:GITHUB_USERNAME
GITHUB_TOKEN=$env:GITHUB_TOKEN
TEST_ENV=$env:TEST_ENV
TEST_SUITE=$env:TEST_SUITE
"@ | Out-File -Encoding ASCII .env

                            docker compose down
                            docker compose up --abort-on-container-exit
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
    }

    post {
        always {
            archiveArtifacts artifacts: 'allure-results/**', fingerprint: true
            archiveArtifacts artifacts: 'allure-report.zip', fingerprint: true
         script {
            bat 'dir allure-report.zip'
            emailext(
                subject: "Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}",
                to: 'debasmita25@gmail.com',
                mimeType: 'text/html',
                attachmentsPattern: '*/allure-report.zip',
                attachLog: true,
                body: """
                <h3>Build Summary</h3>
                <p><b>Job:</b> ${env.JOB_NAME}</p>
                <p><b>Status:</b> ${currentBuild.currentResult}</p>

                <p><a href="${env.BUILD_URL}allure">View Allure Report in Jenkins</a></p>

                <p>Allure HTML report attached.</p>
                <p>Navigate to the download directory and execute the following command to view the report: </p>
                <p><b> allure open allure-report </b> </p>
                """
            )
        }
        }

        success {
            echo "✅ Pipeline Passed"
        }

        failure {
            echo "❌ Pipeline Failed"
        }
    }
}