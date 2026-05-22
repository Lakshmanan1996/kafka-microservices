pipeline {

    agent none

    tools {
        maven 'Maven'
        
        
    }

    environment {

        IMAGE1       = "base-domains"
        IMAGE2       = "email-service"
        IMAGE3       = "order-service"
        IMAGE4       = "stock-service"
        DOCKERHUB_USER = "lakshvar96"
        GIT_REPO = "https://github.com/Lakshmanan1996/kafka-microservices.git"
    }
    
   /* =====================================================   
   CHECKOUT
    ===================================================== */

    stages {

        stage('Checkout Code') {
            agent { label 'workernode1' }
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: 'master']],
                    userRemoteConfigs: [[url: "${GIT_REPO}"]]
                ])
            }
        }


        stage('Stash Source') {
            agent { label 'workernode1' }
            steps {
                stash includes: '**/*', name: 'source-code'
            }
        }


        /* ===================== Build Maven Stage ===================== */
        stage('Build') {
            agent { label 'workernode2' }
            
                    
            steps {
                unstash 'source-code'


                
                dir('base-domains') {
                    sh 'mvn clean package -DskipTests'
                }
                
                
                
                dir('order-service') {
                    sh 'mvn clean package -DskipTests'
                }
                
                dir('stock-service') {
                    sh 'mvn clean package -DskipTests'
                }

                dir('email-service') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        /* =====================================================
           SONARQUBE ANALYSIS
        ===================================================== */

        stage('SonarQube Analysis') {
            agent { label 'workernode2' }
            steps {
                unstash 'source-code'
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    
                    withSonarQubeEnv('sonarqube') {
                        dir('base-domains') {
                        sh """
                             mvn clean verify sonar:sonar \
                             -DskipTests \
                             -Dsonar.projectKey=microservices \
                             -Dsonar.projectName=microservices \
                        """
                        }
                        dir('email-service') {
                        sh """
                             mvn clean verify sonar:sonar \
                             -DskipTests \
                             -Dsonar.projectKey=microservices \
                             -Dsonar.projectName=microservices \
                        """
                        }
                        dir('order-service') {
                        sh """
                             mvn clean verify sonar:sonar \
                             -DskipTests \
                             -Dsonar.projectKey=microservices \
                             -Dsonar.projectName=microservices \
                        """
                        }
                        dir('stock-service') {
                        sh """
                             mvn clean verify sonar:sonar \
                             -DskipTests \
                             -Dsonar.projectKey=microservices \
                             -Dsonar.projectName=microservices \
                        """
                        }
                    }
                }
            }
        }

        /* =====================================================
           QUALITY GATE
        ===================================================== */

        stage('Quality Gate') {
            agent { label 'workernode2' }
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        /* =====================================================
           OWASP DEPENDENCY CHECK
        ===================================================== */

        stage('OWASP Dependency Check') {
            agent { label 'workernode2' }

            steps {
                unstash 'source-code'

                dependencyCheck(
                    odcInstallation: 'OWASP-DC',
                    additionalArguments: '''
                        --scan ${WORKSPACE}
                        --format ALL
                        --out ${WORKSPACE}/dependency-check-report

                        --data /var/jenkins_home/odc-data
                        --noupdate

                        --nvdApiKey YOUR_NVD_API_KEY

                        --exclude **/node_modules/**
                        --exclude **/dist/**
                        --exclude **/target/**
                        --exclude **/.git/**
                    '''
                )   
            }
        }

        /* =====================================================
           DOCKER BUILD
        ===================================================== */

        stage('Docker Build') {
            agent { label 'workernode3' }
            steps {
                unstash 'source-code'
                
                echo "Build a image for base-domains"
                
                dir('base-domains') {
                sh """
                docker build -t ${DOCKERHUB_USER}/${IMAGE1}:${BUILD_NUMBER} .
                docker tag ${DOCKERHUB_USER}/${IMAGE1}:${BUILD_NUMBER} ${DOCKERHUB_USER}/${IMAGE1}:latest 
                """
                }
                
                echo "Build a image for email-service"

                dir('email-service') {
                sh """
                docker build -t ${DOCKERHUB_USER}/${IMAGE2}:${BUILD_NUMBER} .
                docker tag ${DOCKERHUB_USER}/${IMAGE2}:${BUILD_NUMBER} ${DOCKERHUB_USER}/${IMAGE2}:latest 
                """
                }

                echo "Build a image for order-service"

                dir('order-service') {
                sh """
                docker build -t ${DOCKERHUB_USER}/${IMAGE3}:${BUILD_NUMBER} .
                docker tag ${DOCKERHUB_USER}/${IMAGE3}:${BUILD_NUMBER} ${DOCKERHUB_USER}/${IMAGE3}:latest 
                """
                }

                echo "Build a image for stock-service"
                
                dir('stock-service') {
                sh """
                docker build -t ${DOCKERHUB_USER}/${IMAGE4}:${BUILD_NUMBER} .
                docker tag ${DOCKERHUB_USER}/${IMAGE4}:${BUILD_NUMBER} ${DOCKERHUB_USER}/${IMAGE4}:latest 
                """                                                                                       
                }
            }
        }

        /* =====================================================
           TRIVY IMAGE SCAN
        ===================================================== */

        stage('Trivy Scan') {
            agent { label 'workernode3' }
            steps {
                sh """
                trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${IMAGE1}:${BUILD_NUMBER}
                trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${IMAGE2}:${BUILD_NUMBER}
                trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${IMAGE3}:${BUILD_NUMBER}
                trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${IMAGE4}:${BUILD_NUMBER}
                """
            }
        }

         /* =====================================================
           DOCKER PUSH
        ===================================================== */

        stage('Push Image') {
            agent { label 'workernode3' }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                }

                sh """
                docker push ${DOCKERHUB_USER}/${IMAGE1}:${BUILD_NUMBER}
                docker push ${DOCKERHUB_USER}/${IMAGE1}:latest
                """

                 sh """
                docker push ${DOCKERHUB_USER}/${IMAGE2}:${BUILD_NUMBER}
                docker push ${DOCKERHUB_USER}/${IMAGE2}:latest
                """

                sh """
                docker push ${DOCKERHUB_USER}/${IMAGE3}:${BUILD_NUMBER}
                docker push ${DOCKERHUB_USER}/${IMAGE3}:latest
                """

                sh """
                docker push ${DOCKERHUB_USER}/${IMAGE4}:${BUILD_NUMBER}
                docker push ${DOCKERHUB_USER}/${IMAGE4}:latest
                """

                     
            }
        }
    }

    post {
        success {
            echo "✅ kafka-microservices CI Pipeline SUCCESS"
        }
        failure {
            echo "❌ kafka-microservices CI Pipeline FAILED"
        }
    }
}
