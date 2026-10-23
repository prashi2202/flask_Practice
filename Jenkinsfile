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
                    python3 -m venv .venv
                    .venv/bin/python -m pip install --upgrade pip
                    .venv/bin/python -m pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    set -eu
                    python3 -c "import socket; s=socket.create_connection(('127.0.0.1',27017),timeout=3); s.close()"
                    .venv/bin/python -m pytest -v --junitxml=test-results.xml
                '''
            }
            post {
                always {
                    junit testResults: 'test-results.xml', allowEmptyResults: true
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

                    if [ -f staging.pid ]; then
                        old_pid=$(cat staging.pid)
                        if kill -0 "$old_pid" 2>/dev/null; then
                            kill "$old_pid" || true
                            for i in $(seq 1 10); do
                                kill -0 "$old_pid" 2>/dev/null || break
                                sleep 1
                            done
                        fi
                        rm -f staging.pid
                    fi

                    MONGO_URI='mongodb://127.0.0.1:27017/flask_staging_db' \
                    SECRET_KEY='jenkins-local-staging-key' \
                    nohup .venv/bin/python -c \
                      "from app import app; app.run(host='127.0.0.1', port=5001, debug=False)" \
                      > staging.log 2>&1 < /dev/null &

                    echo $! > staging.pid

                    ready=0
                    for i in $(seq 1 30); do
                        if curl --silent --fail http://127.0.0.1:5001/ >/dev/null; then
                            ready=1
                            break
                        fi
                        sleep 2
                    done

                    if [ "$ready" -ne 1 ]; then
                        echo "Staging application did not become ready."
                        cat staging.log
                        exit 1
                    fi

                    echo "Staging deployment is responding on port 5001."
                    curl --silent --show-error --fail \
                      --output /dev/null \
                      --write-out 'HTTP status: %{http_code}\\n' \
                      http://127.0.0.1:5001/
                '''
            }
        }
    }

    post {
        success {
            echo 'Build, tests, and applicable deployment stages succeeded.'
        }
        failure {
            echo 'Pipeline failed. Review the stage logs and test reports.'
        }
        always {
            echo "Pipeline completed with status: ${currentBuild.currentResult}"
        }
    }
}
