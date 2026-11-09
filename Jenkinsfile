
pipeline {
    agent { label "${params.ENVIRONMENT}" }

    environment {
        PROJECT_KEY = 'Pythonproject'
        JFROG_URL = 'https://jfrog.lakshmanakabil.shop/artifactory/Pythonproject'
    }

    parameters {
        string(
            name: 'VERSION',
            defaultValue: '1.0.0',
            description: 'Version of the application'
        )

        booleanParam(
            name: 'RUN_SONAR',
            defaultValue: false,
            description: 'Run SonarQube scan'
        )

        choice(
            name: 'ENVIRONMENT',
            choices: ['Dev', 'staging', 'prod'],
            description: 'Deployment environment'
        )
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/master']],
                    extensions: [],
                    userRemoteConfigs: [[
                        credentialsId: '',
                        url: 'https://github.com/Kabil-148/python-app-cicd-jenkins-pipeline.git'
                    ]]
                )
            }
        }

        stage('Test') {
            steps {
                sh '''
                    set -e

                    python3 -m venv venv
                    . venv/bin/activate

                    pip install -r requirements.txt
                    pip install coverage

                    python -m coverage run --source=app test.py

                    python -m coverage report

                    python -m coverage xml -o coverage.xml
                '''
            }
        }

        stage('SonarQube Analysis') {
            when {
                expression { params.RUN_SONAR }
            }

            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=${PROJECT_KEY} \
                            -Dsonar.sources=app.py,templates,static \
                            -Dsonar.python.coverage.reportPaths=coverage.xml
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            when {
                expression { params.RUN_SONAR }
            }

            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Package and Push') {
            steps {
                sh '''
                    set -e

                    mkdir -p dist

                    zip -qr "dist/python-app-${VERSION}-${BUILD_NUMBER}.zip" \
                    app.py requirements.txt templates static

                    if [ -d Images ]; then
                        zip -qr "dist/python-app-${VERSION}-${BUILD_NUMBER}.zip" Images
                    fi
                '''

                archiveArtifacts artifacts: 'dist/*.zip', fingerprint: true

                withCredentials([
                    usernamePassword(
                        credentialsId: 'Jrog_cred',
                        usernameVariable: 'username',
                        passwordVariable: 'password'
                    )
                ]) {
                    sh '''
                        set +x

                        curl --fail --silent --show-error --retry 2 \
                        -u "$username:$password" \
                        -T "dist/python-app-${VERSION}-${BUILD_NUMBER}.zip" \
                        "${JFROG_URL}/python-app-${VERSION}-${BUILD_NUMBER}.zip"
                    '''
                }
            }
        }

        stage('Deploy') {
            when {
                expression { params.ENVIRONMENT == 'Dev' }
            }

            steps {
                sh '''
                    set -e

                    mkdir -p /home/ubuntu/python-app-jenkins

                    unzip -oq "dist/python-app-${VERSION}-${BUILD_NUMBER}.zip" \
                    -d /home/ubuntu/python-app-jenkins

                    sudo -n systemctl restart python-app-jenkins

                    sleep 3

                    curl --fail --silent --show-error \
                    http://127.0.0.1:5000/
                '''
            }
        }
    }
}
