pipeline {

    // Run the pipeline on the available Jenkins machine.
    agent any

    options {
        // Add the time beside every Jenkins log entry.
        timestamps()

        // Stop the complete pipeline if it exceeds 40 minutes.
        timeout(time: 40, unit: 'MINUTES')

        // Keep only the latest 20 Jenkins builds.
        buildDiscarder(logRotator(numToKeepStr: '20'))

        // Prevent Jenkins from performing an additional automatic checkout.
        skipDefaultCheckout(true)
    }

    environment {
        // Full location of the Python installation on the laptop.
        SYSTEM_PYTHON = 'C:\\Users\\riteshindupur\\miniconda3\\python.exe'

        // Minimum acceptable classifier accuracy.
        EVAL_THRESHOLD = '0.85'

        // Python executable inside the project virtual environment.
        PY = 'venv\\Scripts\\python.exe'

        // Secure AI key stored in Jenkins Credentials.
        LLM_API_KEY = credentials('llm-api-key-Sep05')

        // Local folder acting as the staging server.
        STAGING = 'C:\\stx\\staging'
    }

    stages {

        stage('1. Checkout') {
            steps {
                echo 'Fetching the exact commit that triggered this build...'

                checkout scm

                bat '''
                    git rev-parse --short HEAD > commit.txt
                    type commit.txt
                '''
            }
        }

        stage('2. Environment') {
            steps {
                echo 'Building a clean, isolated Python environment...'

                bat '''
                    if exist venv rmdir /s /q venv
                    if not exist reports mkdir reports
                    if not exist dist mkdir dist

                    "%SYSTEM_PYTHON%" --version
                    "%SYSTEM_PYTHON%" -m venv venv
                    "%PY%" -m pip install --upgrade pip
                    "%PY%" -m pip install -r requirements.txt
                '''
            }
        }

        stage('3. Lint') {
            steps {
                echo 'Checking code style and obvious errors...'

                bat '''
                    "%PY%" -m ruff check app tests evals
                '''
            }
        }

        stage('4. Unit Tests') {
            steps {
                echo 'Running unit tests and API smoke tests...'

                bat '''
                    "%PY%" -m pytest tests -q --junitxml=reports/junit.xml
                '''
            }
        }

        stage('5. Evaluation Gate') {
            steps {
                echo 'Measuring classifier quality against the labelled dataset...'

                bat '''
                    "%PY%" evals\\run_eval.py --threshold %EVAL_THRESHOLD%
                '''
            }
        }

        stage('6. Package') {
            steps {
                echo 'Producing a versioned, shippable ZIP file...'

                bat '''
                    if not exist dist mkdir dist
                    tar -a -c -f dist\\triage-%BUILD_NUMBER%.zip app requirements.txt
                    dir dist
                '''
            }
        }

        stage('7. Approval') {
            steps {
                script {
                    timeout(time: 15, unit: 'MINUTES') {
                        input(
                            message: 'Deploy this build to Strataxis staging?',
                            ok: 'Approve deployment'
                        )
                    }
                }
            }
        }

        stage('8. Deploy to Staging') {
            steps {
                echo 'Releasing the approved artifact to staging...'

                bat '''
                    if not exist "%STAGING%" mkdir "%STAGING%"
                    copy /Y "dist\\triage-%BUILD_NUMBER%.zip" "%STAGING%\\"
                    echo %DATE% %TIME% build %BUILD_NUMBER% >> "%STAGING%\\log.txt"
                '''
            }
        }
    }

    post {
        always {
            echo 'Saving available test results and build artifacts...'

            junit(
                testResults: 'reports/junit.xml',
                allowEmptyResults: true
            )

            archiveArtifacts(
                artifacts: 'dist/*.zip, reports/*, commit.txt',
                allowEmptyArchive: true,
                fingerprint: true
            )
        }

        success {
            echo 'GREEN: this commit is safe to show the client.'
            echo 'The approved package was deployed to staging.'
        }

        failure {
            echo 'RED: something failed. Nothing was deployed.'
        }

        aborted {
            echo 'ABORTED: the build or approval was cancelled.'
        }
    }
}
