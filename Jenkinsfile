pipeline {

   agent any

   parameters{
    choice(name : 'TEST_ENV', choices : ['qa','stage','prod'],description ='Select the Test Environment')
    string(name : 'TEST_SUITE',defaultValue : 'testng.xml',description:'Enter the TestNG suite file')
   }

   environment {
    TEST_IMAGE='debasmita25/github-api-tests:latest'
   }



   stages {

    stage('Checkout Code') {
            steps {
                checkout scm
            }
    } 


    stage('pull Test Image '){

        script{

            if(isUnix())
            {
                sh "docker pull ${TEST_IMAGE}"
            } else
            {
                bat "docker pull %TEST_IMAGE%"
            }
        }
        
    }
   }

    stage('Pre-Clean SetUp'){

        script{

            if(isUnix())
            {
              sh '''
                rm -rf allure-results allure-report || true
                mkdir -p allure-results allure-report
                '''
            }
            else {

                bat '''
                IF EXIST allure-results rmdir /s /q allure-results
                IF EXIST allure-report rmdir /s /q allure-report

                mkdir allure-results
                mkdir allure-report
                '''

            }
        }
    }

   

    stage('Run API Tests')
    {
         script {
            withCredentials([usernamePassword(
                credentialsId: 'api-creds',
                usernameVariable: 'GITHUB_USERNAME',
                passwordVariable: 'GITHUB_TOKEN'
            )]) {

            if(isUnix())
            {
               // 🔹 Linux Execution
                    sh """
                    set -e

                    export GITHUB_USERNAME=$GITHUB_USERNAME
                    export GITHUB_TOKEN=$GITHUB_TOKEN
                    export TEST_ENV=${params.TEST_ENV}
                    export TEST_SUITE=${params.TEST_SUITE}

                    docker compose down || true
                    docker compose up --abort-on-container-exit
                    """
            }
            else
            {
                // 🔹 Windows Execution (PowerShell)
                    powershell """
                    \$ErrorActionPreference = "Stop"

                    \$env:GITHUB_USERNAME = "${GITHUB_USERNAME}"
                    \$env:GITHUB_TOKEN    = "${GITHUB_TOKEN}"
                    \$env:TEST_ENV=${params.TEST_ENV}
                    \$env:TEST_SUITE=${params.TEST_SUITE}

                    try {
                        docker compose down
                    } catch {
                        Write-Host "Ignoring cleanup error"
                    }

                    docker compose up --abort-on-container-exit
                    """
               
            }

        }
    }
   }

    stage('Post Cleanup (Containers)') {
            steps {
                if(isUnix())
                    sh 'docker compose down || true'
                else
                    bat 'docker compose down || true'    
            }
        }

    stage('Check if Allure Results exist') {
            steps {
               script {
                if (isUnix()) {
                    sh '''
                    echo "Checking allure-results directory..."

                    if [ -d "allure-results" ]; then
                        echo "Directory exists"
                        ls -la allure-results
                    else
                        echo "Directory does NOT exist"
                        exit 1
                    fi
                    '''
                } else {
                    powershell '''
                    Write-Host "Checking allure-results directory..."

                    if (Test-Path "allure-results") {
                        Write-Host "Directory exists"
                        Get-ChildItem "allure-results"
                    } else {
                        Write-Host "Directory does NOT exist"
                        exit 1
                    }
                    '''
                }
    }}
    }
  stage('Allure Report') {
            steps {
                allure([
                    results: [[path: 'allure-results']]
                ])
            }
        }
    

    post {
        always {
            archiveArtifacts artifacts: 'allure-results/**', fingerprint: true
        }
    

        success {
            echo "✅ Pipeline Passed"
        }

        failure {
            echo "❌ Pipeline Failed"
        }
    }
}
