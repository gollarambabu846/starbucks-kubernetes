pipeline {
    agent any
    
    tools {
        jdk 'jdk'
        nodejs 'node17'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Clean Workspace') {
            steps { cleanWs() }
        }

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/gollarambabu846/starbucks-kubernetes.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=starbucks \
                    -Dsonar.projectKey=starbucks
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh '''
                        docker build -t starbucks .
                        docker tag starbucks gollarambabu/starbucks:latest
                        docker push gollarambabu/starbucks:latest
                        '''
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image gollarambabu/starbucks:latest > trivyimage.txt'
            }
        }

        stage('Deploy to Docker') {
            steps {
                sh '''
                docker rm -f starbucks || true
                docker run -d --name starbucks -p 4000:4000 gollarambabu/starbucks:latest
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                dir('kubernetes') {
                    sh '''
                    aws eks update-kubeconfig --region ap-south-1 --name rambabu
                    kubectl apply -f manifest.yml
                    kubectl get pods
                    kubectl get svc
                    '''
                }
            }
        }
    }

    post {
        always {
            emailext(
                subject: "Pipeline ${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build URL: ${env.BUILD_URL}",
                to: 'gollarambabu70@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
        }
    }
}