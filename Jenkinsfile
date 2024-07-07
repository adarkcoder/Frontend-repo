pipeline {
    agent any
    
    tools {
        nodejs 'NodeJS 14.x'
    }

    environment {
        AWS_CREDENTIALS_ID = 'aws-credentials-id'
        DOCKER_IMAGE = 'public.ecr.aws/e8n4i2w8/frontend:latest'
        AWS_REGION = 'us-east-1'
        DEPLOYMENT_NAME = "frontend"
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/adarkcoder/Frontend-repo.git'
            }
        }
        
        stage('Install Dependencies') {
            steps {
                dir('FullStackAppFrontEnd') {
                    sh 'npm install'
                }
            }
        }

        stage('Test') {
            steps {
                // Run Maven clean and test phases with error handling
                dir('FullStackAppFrontEnd') {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        sh 'npm test'
                    }
                }
            }
        }

        stage('SonarQube - SAST') {
            environment{
                scannerHome = tool 'sonar-scanner'
            }
            steps {
                // Run SonarQube analysis after build and test
                dir('FullStackAppFrontEnd') {
                    script {
                        withSonarQubeEnv('Sonar') {
                            sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=Frontend-project \
                                  -Dsonar.projectName=Frontend-project \
                                  -Dsonar.sources=src/ \
                                  -Dsonar.tests=src/ \
                                  -Dsonar.test.inclusions=src/**/*.test.js \
                                  -Dsonar.clover.reportPaths=coverage/clover.xml \
                                  -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info '''      
                        }
                    }
                }
                timeout(time: 2, unit: 'MINUTES') {
                    script {
                        try {
                            waitForQualityGate abortPipeline: true
                        } catch (err) {
                            echo "Quality Gate check failed: ${err.message}"
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                // Build Docker image within FullStackApp directory
                dir('FullStackAppFrontEnd') {
                    script {
                        sh 'docker build -t ${DOCKER_IMAGE} .'
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: AWS_CREDENTIALS_ID, usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh '''
                        aws ecr-public get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin public.ecr.aws/e8n4i2w8
                        docker push ${DOCKER_IMAGE}
                        '''
                    }
                }
            }
        }
        
        // stage('Deploy to Kubernetes') {
        //     steps {
        //         dir('Frontend') {
        //             script {
        //                 // Use kubeconfig to configure Kubernetes context
        //                 withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: '', contextName: '', credentialsId: 'kube-config', namespace: '', serverUrl: '']]) {
        //                     withCredentials([usernamePassword(credentialsId: AWS_CREDENTIALS_ID, usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
        //                         sh 'kubectl apply -f .'
        //                         sh 'kubectl rollout restart deploy ${frontend}'
        //                     }
        //                 }
        //             }
        //         }
        //      }
        // }
       stage('Deploy and Rollout') {
            parallel {
                stage('Deploy to Kubernetes') {
                    steps {
                        dir('Frontend') {
                            script {
                                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: '', contextName: '', credentialsId: 'kube-config', namespace: '', serverUrl: '']]) {
                                    withCredentials([usernamePassword(credentialsId: AWS_CREDENTIALS_ID, usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                                        sh 'kubectl apply -f .'
                                    }
                                }
                            }
                        }
                    }
                }

                stage('k8s Rollout') {
                    steps {
                        script {
                                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: '', contextName: '', credentialsId: 'kube-config', namespace: '', serverUrl: '']]) {
                                    withCredentials([usernamePassword(credentialsId: AWS_CREDENTIALS_ID, usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                                        sh 'bash k8RolloutStage.sh'
                                    }
                                }
                            }
                    }
                }
            }
       }
        
    }

    post {
        always {
            // Clean up any resources or perform cleanup tasks
            cleanWs()
        }
    }
}
