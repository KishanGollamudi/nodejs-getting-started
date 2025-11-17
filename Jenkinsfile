pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "kishangollamudi/nodeapp"
        VERSION      = "${env.BUILD_NUMBER}"
        NEXUS_URL    = "http://54.85.207.105:8081/repository/nodejs"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                echo "Running npm install..."
                // keep this simple — runs using node/npm present in the Jenkins agent container
                sh "npm install --no-audit --no-fund"
            }
        }

        stage('Run Tests') {
            steps {
                echo "Skipping tests (adjust if you want to run them)..."
                sh 'echo "Tests skipped"'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "Running SonarQube analysis..."
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('My-Sonar') {
                        script {
                            // Try to use the Jenkins SonarScanner installation if present; otherwise call sonar-scanner
                            try {
                                def scannerHome = tool name: 'SonarScanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                                sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=nodeapp -Dsonar.sources=. -Dsonar.login=${SONAR_TOKEN}"
                            } catch (err) {
                                echo "SonarScanner tool not found in Jenkins tools; falling back to 'sonar-scanner' in PATH"
                                sh "sonar-scanner -Dsonar.projectKey=nodeapp -Dsonar.sources=. -Dsonar.login=${SONAR_TOKEN}"
                            }
                        }
                    }
                }
            }
        }

        stage('Package Artifact (tar.gz)') {
            steps {
                echo "Creating deterministic tar.gz from current commit using git-archive..."
                // git archive creates a clean, consistent archive (no 'file changed' race)
                sh "git archive --format=tar.gz -o nodeapp-${VERSION}.tar.gz HEAD"
            }
        }

        stage('Upload to Nexus') {
            steps {
                echo "Uploading artifact to Nexus (raw repo) ..."
                withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh """
                        curl -v -u $NEXUS_USER:$NEXUS_PASS \
                          --fail --show-error \
                          --upload-file nodeapp-${VERSION}.tar.gz \
                          ${NEXUS_URL}/nodeapp-${VERSION}.tar.gz
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
                echo "Pushing Docker image to Docker Hub..."
                withCredentials([usernamePassword(credentialsId: 'dockerhub-user', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh """
                        echo $DH_PASS | docker login -u $DH_USER --password-stdin
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
