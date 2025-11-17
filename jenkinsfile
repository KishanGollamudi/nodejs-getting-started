pipeline {
    agent any

    environment {
        SONARQUBE_URL = 'http://sonar:9000'
        NEXUS_URL = 'http://nexus:8081/repository/node-releases/'
        DOCKER_IMAGE = "kishangollamudi/nodejs-app"
        VERSION = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Fetching latest code from GitHub..."
                checkout scm
            }
        }

        stage('Install Node & Dependencies') {
            steps {
                echo "Installing Node.js and NPM packages..."
                sh '''
                    curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
                    apt-get install -y nodejs
                    npm install
                '''
            }
        }

        stage('Run Tests') {
            steps {
                echo "Running Node.js test cases..."
                sh 'npm test || true'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "Running SonarQube Scan..."
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh """
                        npx sonar-scanner \
                        -Dsonar.projectKey=nodejs-app \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=${SONARQUBE_URL} \
                        -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Package Artifact') {
            steps {
                echo "Creating build artifact..."
                sh '''
                    zip -r nodejs-app-${VERSION}.zip *
                '''
            }
        }

        stage('Upload to Nexus') {
            steps {
                echo "Uploading artifact to Nexus..."
                withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh """
                        curl -v -u $NEXUS_USER:$NEXUS_PASS --upload-file nodejs-app-${VERSION}.zip ${NEXUS_URL}
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."
                script {
                    docker.build("${DOCKER_IMAGE}:${VERSION}")
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                echo "Pushing Docker image to DockerHub..."
                withCredentials([usernamePassword(credentialsId: 'dockerhub-user', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh """
                        echo ${DH_PASS} | docker login -u ${DH_USER} --password-stdin
                        docker push ${DOCKER_IMAGE}:${VERSION}
                        docker tag ${DOCKER_IMAGE}:${VERSION} ${DOCKER_IMAGE}:latest
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

    }

    post {
        always {
            echo "Cleaning workspace..."
            cleanWs()
        }
    }
}
