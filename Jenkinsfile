pipeline {
    agent any

    tools {
        jdk 'jdk'
        nodejs 'node'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        AWS_DEFAULT_REGION = 'ap-south-1'
        CLUSTER_NAME = 'poc-eks-cluster-1'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Hotstar \
                        -Dsonar.projectKey=Hotstar
                    '''
                }
            }
        }

        /* -------- QUALITY GATE SKIPPED (NON-BLOCKING) -------- */
        stage('Quality Gate') {
            steps {
                echo "Skipping SonarQube Quality Gate wait"
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey 7D78E0BF-88EF-F011-8366-0EBF96DE670D',
                                odcInstallation: 'DC'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('TRIVY FS SCAN') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh '''
                            docker build -t hotstar .
                            docker tag hotstar danielbernard/hotstar:2
                            docker push danielbernard/hotstar:2
                        '''
                    }
                }
            }
        }

        stage('TRIVY IMAGE SCAN') {
            steps {
                sh 'trivy image danielbernard/hotstar:2 > trivyimage.txt'
            }
        }

        stage('Configure Kubeconfig for EKS') {
            steps {
                sh '''
                    aws eks update-kubeconfig \
                      --region $AWS_DEFAULT_REGION \
                      --name $CLUSTER_NAME
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    echo "Deploying application to EKS..."
                    kubectl apply -f K8S/
                '''
            }
        }
    }

    post {
        always {
            script {
                def buildStatus = currentBuild.currentResult
                def buildUser = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId ?: 'Github User'

                emailext (
                    subject: "Pipeline ${buildStatus}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <p>Jenkins HOTSTAR CICD pipeline status</p>
                        <p>Project: ${env.JOB_NAME}</p>
                        <p>Build Number: ${env.BUILD_NUMBER}</p>
                        <p>Build Status: ${buildStatus}</p>
                        <p>Triggered by: ${buildUser}</p>
                        <p>Build URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    """,
                    to: 'bernarddaniel171@gmail.com',
                    from: 'bernarddaniel171@gmail.com',
                    replyTo: 'bernarddaniel171@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
                )
            }
        }
    }
}
