pipeline {
    agent any

    parameters {
        choice(name: 'TEST_ENV', choices: ['qa','stage','prod'], description: 'Environment')
        string(name: 'TEST_SUITE', defaultValue: 'testng.xml', description: 'TestNG Suite')
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Run Tests (Docker)') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'api-creds',
            usernameVariable: 'GITHUB_USERNAME',
            passwordVariable: 'GITHUB_TOKEN'
        )]) {
            sh '''
                export TEST_ENV=${TEST_ENV}
                export TEST_SUITE=${TEST_SUITE}
                export GITHUB_USERNAME=${GITHUB_USERNAME}
                export GITHUB_TOKEN=${GITHUB_TOKEN}

                # Pull latest image from Docker Hub
                docker pull debasmita25/github-api-tests:latest

                # Run tests via docker-compose
                docker compose down || true
                docker compose up --abort-on-container-exit
            '''
        }
    }
}
    }

    post {
        always {
            archiveArtifacts artifacts: 'allure-results/**', fingerprint: true
            archiveArtifacts artifacts: 'allure-report.zip', fingerprint: true

            emailext(
                subject: "Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}",
                to: 'debasmita25@gmail.com',
                mimeType: 'text/html',
                attachLog: true,
                body: """
                <h3>Build Summary</h3>
                <p><b>Status:</b> ${currentBuild.currentResult}</p>
                <p><a href="${env.BUILD_URL}artifact/allure-report.zip">Download Allure Report</a></p>
                """
            )
        }
    }
}