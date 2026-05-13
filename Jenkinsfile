pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    extensions: [],
                    userRemoteConfigs: [[
                        credentialsId: 'gitcred',
                        url: 'https://github.com/Vijayasaka-112/Book-My-Show.git'
                    ]]
                )

                sh 'ls -la'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                    cd bookmyshow-app

                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=BMS \
                    -Dsonar.projectKey=BMS \
                    -Dsonar.sources=.
                    '''
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                cd bookmyshow-app

                if [ -f package.json ]; then
                    rm -rf node_modules package-lock.json
                    npm install
                else
                    echo "package.json not found!"
                    exit 1
                fi
                '''
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {

                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin

                    docker build --no-cache \
                    -t vijaya123qw/bms:latest \
                    -f bookmyshow-app/Dockerfile bookmyshow-app

                    docker push vijaya123qw/bms:latest
                    '''
                }
            }
        }

        stage('Deploy to Container') {
            steps {
                sh '''
                docker stop bms || true
                docker rm bms || true

                docker run -d \
                --restart=always \
                --name bms \
                -p 3000:3000 \
                vijaya123qw/bms:latest

                sleep 5

                docker ps -a
                docker logs bms
                '''
            }
        }
    }

    post {
        always {
            emailext(
                attachLog: true,
                subject: "${currentBuild.result}",
                body: "Project: ${env.JOB_NAME}<br/>Build Number: ${env.BUILD_NUMBER}<br/>URL: ${env.BUILD_URL}",
                to: 'vijayasaka29@gmail.com',
                attachmentsPattern: 'trivyfs.txt'
            )
        }
    }
}
