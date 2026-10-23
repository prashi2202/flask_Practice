
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 20, unit: 'MINUTES')
    }

    environment {
        MONGO_URI = 'mongodb://127.0.0.1:27017/test_student_db'
        SECRET_KEY = 'jenkins-local-test-key'
        STAGING_PORT = '5001'
    }

    stages {
        stage('Build') {
            steps {
                sh '''
                    set -eu

                    echo "Creating Python virtual environment..."
                    python3 -m venv .venv

                    echo "Upgrading pip..."
                    .venv/bin/python -m pip install --upgrade pip

                    echo "Installing application dependencies..."
                    .venv/bin/python -m pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    set -eu

                    echo "Checking MongoDB connectivity..."
                    python3 -c "import socket; s=socket.create_connection(('127.0.0.1',27017),timeout=3); s.close()"

                    echo "Running Flask application tests..."
                    .venv/bin/python -m pytest -v --junitxml=test-results.xml
                '''
            }

            post {
                always {
                    junit(
                        testResults: 'test-results.xml',
                        allowEmptyResults: true
                    )
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                expression {
                    env.BRANCH_NAME == 'main' ||
                    env.GIT_BRANCH == 'origin/main' ||
                    env.GIT_BRANCH == 'refs/remotes/origin/main'
                }
            }

            steps {
                sh '''
                    set -eu

                    echo "Deploying Flask application to staging..."
                    sudo -n /usr/local/sbin/deploy-flask-staging

                    echo "Verifying staging endpoint..."
                    curl --silent --show-error --fail \
                        --output /dev/null \
                        --write-out 'HTTP status: %{http_code}\\n' \
                        http://127.0.0.1:5001/

                    echo "Staging deployment verified."
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Build, tests, and applicable deployment stages passed.'
        }

        failure {
            echo 'FAILURE: Review the failed stage and Console Output.'
        }

        always {
            echo "Pipeline completed with status: ${currentBuild.currentResult}"
        }
    }
}
