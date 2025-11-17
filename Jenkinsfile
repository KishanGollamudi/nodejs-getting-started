pipeline {
    agent any

    environment {
        SONAR_TOKEN = credentials('sonar-token')
        NEXUS_CRED  = credentials('nexus')
        DOCKER_HUB  = credentials('dockerhub-user')
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out code..."
                git branch: 'main', url: 'https://github.com/KishanGollamudi/nodejs-getting-started.git'
            }
        }

        stage('Install & Build App') {
            steps {
                sh '''
                    apt-get update && apt-get install -y zip

                    npm install --no-audit --no-fund

                    echo "No build step for this project, skipping build..."
                    
                    zip -r artifact.zip \
                        . \
                        --exclude="**/node_modules/**"
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('My-Sonar') {
                    sh '''
                        export PATH=$PATH:/opt/sonar-scanner/bin

                        sonar-scanner \
                        -Dsonar.projectKey=nodeapp \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://54.85.207.105:9000 \
                        -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('Upload Artifact to Nexus') {
            steps {
                sh '''
                    curl -u $NEXUS_CRED_USR:$NEXUS_CRED_PSW \
                    --upload-file artifact.zip \
                    http://54.85.207.105:8081/repository/nodejs/artifact-${BUILD_NUMBER}.zip
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                    --build-arg NEXUS_URL=http://54.85.207.105:8081 \
                    --build-arg REPO_PATH=repository/nodejs/artifact-${BUILD_NUMBER}.zip \
                    --build-arg NEXUS_USER=$NEXUS_CRED_USR \
                    --build-arg NEXUS_PASS=$NEXUS_CRED_PSW \
                    -t kishangollamudi/nodeapp:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh '''
                    echo $DOCKER_HUB_PSW | docker login -u $DOCKER_HUB_USR --password-stdin
                    docker push kishangollamudi/nodeapp:${BUILD_NUMBER}
                    docker tag kishangollamudi/nodeapp:${BUILD_NUMBER} kishangollamudi/nodeapp:latest
                    docker push kishangollamudi/nodeapp:latest
                '''
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished!'
        }
    }
}
