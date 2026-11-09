pipeline {
    agent { label "${params.environment}" }
    parameters {
        choice choices: ['dev', 'qa', 'uat', 'prod'], description: 'Choose the env', name: 'environment'
        booleanParam defaultValue: true, description: 'Enable to run sonar scan', name: 'sonar'
        
    }
    
    environment {
        SONARQUBE_ENV = 'sonarscanner' 
        PROJECT_KEY = 'python-app-jenkins'
        JFROG_URL = 'http://44.201.128.89:8081/artifactory/python-app-jenkins/'
    }

    stages {
        stage('checkout') {
            steps {
                checkout scmGit(branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: '', url: 'https://github.com/PrayogReddy/python-app-cicd-jenkins-pipeline.git']])
            }
        }
        stage('Test') {
            steps {
                sh 'python3 -m coverage run --source=app test.py'
                sh 'python3 -m coverage xml -o coverage.xml'
            }
        }
        stage('sonarscan') {
            when {
                expression { params.sonar }
            }
            steps {
                script {
            def scannerHome = tool 'SonarScanner'
            withSonarQubeEnv("${SONARQUBE_ENV}") {
                sh "\"${scannerHome}/bin/sonar-scanner\" -Dsonar.projectKey=${PROJECT_KEY} -Dsonar.sources=app.py,templates,static -Dsonar.python.coverage.reportPaths=coverage.xml"
                }
              }
            }
        }
        stage('quality gate') {
            when {
                expression { params.sonar }
            }
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }
        stage('Package and Push') {
            steps {
                sh 'mkdir -p dist'
                sh 'zip -qr "dist/python-app-${BUILD_NUMBER}.zip" app.py requirements.txt templates static'
                sh 'if [ -d Images ]; then zip -qr "dist/python-app-${BUILD_NUMBER}.zip" Images; fi'

                archiveArtifacts artifacts: 'dist/*.zip', fingerprint: true

                withCredentials([usernamePassword(credentialsId: 'jfrog_creds', usernameVariable: 'JFROG_USER', passwordVariable: 'JFROG_PASSWORD')]) {
                    sh 'set +x; curl --fail --silent --show-error --retry 2 -u "$JFROG_USER:$JFROG_PASSWORD" -T "dist/python-app-${BUILD_NUMBER}.zip" "${JFROG_URL}/python-app-${BUILD_NUMBER}.zip"'
                }
            }                
        }
        stage('Deploy') {
          when {
                  expression { params.environment == 'dev' }
              }
          steps {
                  sh 'mkdir -p /home/ubuntu/python-app-jenkins'
                  sh 'unzip -oq "dist/python-app-${BUILD_NUMBER}.zip" -d /home/ubuntu/python-app-jenkins'
                  sh 'sudo -n systemctl restart python-app-jenkins'
                  sh 'sleep 3'
                  sh 'curl --fail --silent --show-error http://127.0.0.1:5000/'
                }
          }
      }
}
