pipeline {
    agent any
    
    triggers {
        githubPush()   // Auto-trigger on GitHub push
    }

    tools {
        jdk "jdk"
        maven "maven"
    }

    environment {
        // Git
        GIT_REPO_URL = 'https://github.com/aayushITT/Boardgame.git'
        GIT_BRANCH   = 'main'
        
        // App & DockerHub
        APP_NAME   = "blogging-app"
        IMAGE_NAME = "aayush292003/${APP_NAME}"
        VERSION_TAG = "${env.BUILD_NUMBER}"
        FULL_IMAGE  = "${IMAGE_NAME}:${VERSION_TAG}"
        
        // Kubernetes
        K8S_NAMESPACE = "webapps"
        DEPLOYMENT_NAME = "bloggingapp-deployment"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${GIT_BRANCH}", url: "${GIT_REPO_URL}"
            }
        }

        stage('Build (Maven)') {
            steps {
                sh 'mvn -B -ntp clean package -DskipTests'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn -B -ntp test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqubeServer') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh """
                            mvn sonar:sonar \
                              -Dsonar.projectKey=${APP_NAME} \
                              -Dsonar.projectName=${APP_NAME} \
                              -Dsonar.java.binaries=target/classes \
                              -Dsonar.login=$SONAR_TOKEN
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                        echo "Quality Gate passed: ${qg.status}"
                    }
                }
            }
        }

        stage('Publish Artifacts') {
            steps {
                withMaven(
                    globalMavenSettingsConfig: '47981867-14b5-4841-9f3e-30286e7fd42e',
                    jdk: 'jdk',
                    maven: 'maven',
                    traceability: true
                ) {
                    sh "mvn deploy"
                }
            }
        }

        stage('Docker Build & Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                        sh """
                            docker build --pull \
                              -t ${FULL_IMAGE} \
                              -t ${IMAGE_NAME}:latest \
                              .
                        """
                    }
                }
            }
        }

        stage('Docker Push Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                        sh "docker push ${FULL_IMAGE}"
                        sh "docker push ${IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Deploy to K8s') {
            steps {
                withKubeCredentials(kubectlCredentials: [[
                    caCertificate: '',
                    clusterName: 'test-cluster',
                    contextName: '',
                    credentialsId: 'K8s-token',
                    namespace: "${K8S_NAMESPACE}",
                    serverUrl: 'https://B33AAF2FAFEBB286766B97890E4361FC.yl4.ap-south-1.eks.amazonaws.com'
                ]]) {
                    sh """
                        # Update deployment with new image
                        kubectl set image deployment/${DEPLOYMENT_NAME} ${APP_NAME}=${FULL_IMAGE} -n ${K8S_NAMESPACE} --record
                        
                        # Wait for rollout
                        kubectl rollout status deployment/${DEPLOYMENT_NAME} -n ${K8S_NAMESPACE}
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeCredentials(kubectlCredentials: [[
                    caCertificate: '',
                    clusterName: 'test-cluster',
                    contextName: '',
                    credentialsId: 'K8s-token',
                    namespace: "${K8S_NAMESPACE}",
                    serverUrl: 'https://B33AAF2FAFEBB286766B97890E4361FC.yl4.ap-south-1.eks.amazonaws.com'
                ]]) {
                    sh "kubectl get pods -n ${K8S_NAMESPACE} -o wide"
                    sh "kubectl get svc -n ${K8S_NAMESPACE} -o wide"
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline completed. Cleaning up..."
        }
        cleanup {
            sh """
                docker rmi ${FULL_IMAGE} ${IMAGE_NAME}:latest || true
                docker image prune -f || true
            """
        }
    }
}
