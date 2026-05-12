pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'node23'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME = "kastrov/bms:latest"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main',
                credentialsId: 'git',
                url: 'https://github.com/Kondavenkat035/Book-My-Show.git'

                sh 'ls -la'
            }
        }

        stage('Check Installed Tools') {
            steps {
                sh '''
                echo "Java Version:"
                java -version

                echo "Node Version:"
                node -v

                echo "NPM Version:"
                npm -v

                echo "Docker Version:"
                docker --version

                echo "Trivy Version:"
                trivy --version

                echo "Sonar Scanner Version:"
                $SCANNER_HOME/bin/sonar-scanner --version
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {

                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=BMS \
                    -Dsonar.projectKey=BMS
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }

        stage('Install Dependencies') {
            steps {

                dir('bookmyshow-app') {

                    sh '''
                    echo "Checking package.json..."

                    ls -la

                    if [ -f package.json ]; then

                        echo "Removing old node modules..."
                        rm -rf node_modules package-lock.json

                        echo "Cleaning npm cache..."
                        npm cache clean --force

                        echo "Installing dependencies..."
                        npm install

                    else
                        echo "package.json not found!"
                        exit 1
                    fi
                    '''
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {

                dependencyCheck(
                    additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit',
                    odcInstallation: 'DP-Check'
                )

                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy File System Scan') {
            steps {

                sh '''
                trivy fs . > trivyfs.txt
                '''
            }
        }

        stage('Docker Build') {
            steps {

                sh '''
                echo "Building Docker image..."

                docker build --no-cache \
                -t $IMAGE_NAME \
                -f bookmyshow-app/Dockerfile \
                bookmyshow-app
                '''
            }
        }

        stage('Trivy Docker Image Scan') {
            steps {

                sh '''
                trivy image $IMAGE_NAME > trivyimage.txt
                '''
            }
        }

        stage('Docker Push') {
            steps {

                script {

                    withDockerRegistry(
                        credentialsId: 'docker-cred',
                        toolName: 'docker'
                    ) {

                        sh '''
                        echo "Pushing Docker image..."
                        docker push $IMAGE_NAME
                        '''
                    }
                }
            }
        }

        stage('Deploy Container') {
            steps {

                sh '''
                echo "Stopping old container..."

                docker stop bms || true
                docker rm bms || true

                echo "Removing unused Docker images..."
                docker system prune -f

                echo "Running new container..."

                docker run -d \
                --name bms \
                --restart=always \
                -p 3000:3000 \
                $IMAGE_NAME

                echo "Checking running containers..."
                docker ps -a

                echo "Waiting for app startup..."
                sleep 15

                echo "Container logs:"
                docker logs bms
                '''
            }
        }
    }

    post {

        success {

            emailext(
                attachLog: true,
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """
                <h2>Build Success</h2>

                <p><b>Project:</b> ${env.JOB_NAME}</p>
                <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                <p><b>Status:</b> SUCCESS</p>
                <p><b>Build URL:</b> ${env.BUILD_URL}</p>
                """,
                to: 'kondavenkat035@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
        }

        failure {

            emailext(
                attachLog: true,
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """
                <h2>Build Failed</h2>

                <p><b>Project:</b> ${env.JOB_NAME}</p>
                <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                <p><b>Status:</b> FAILED</p>
                <p><b>Build URL:</b> ${env.BUILD_URL}</p>
                """,
                to: 'kondavenkat035@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
        }

        always {

            sh '''
            echo "Pipeline execution completed."
            '''
        }
    }
}
