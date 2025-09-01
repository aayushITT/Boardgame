pipeline {
    agent any
    tools {
        jdk "jdk"
        maven "maven"
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME = "aayush292003/blogging-app"
        IMAGE_TAG = "${env.BUILD_NUMBER}" // unique tag for every build
        FULL_IMAGE = "${IMAGE_NAME}:${IMAGE_TAG}"
    }
    
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/aayushITT/Boardgame.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn clean compile"
            }
        }
        
        stage('Trivy FS') {
            steps {
                sh "trivy fs . --format table -o fs.html"
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqubeServer') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh """
                            $SCANNER_HOME/bin/sonar-scanner \
                              -Dsonar.projectName=Blogging-app \
                              -Dsonar.projectKey=Blogging-app \
                              -Dsonar.java.binaries=target \
                              -Dsonar.login=$SONAR_TOKEN
                        """
                    }
                }
            }
        }
        
        stage('Build & Package') {
            steps {
                // Ensure fresh jar
                sh "mvn clean package -DskipTests"
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
                        sh "docker build --no-cache -t ${FULL_IMAGE} ."
                    }
                }
            }
        }
        
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --format table -o image.html ${FULL_IMAGE}"
            }
        }
        
        stage('Docker Push Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                        sh "docker push ${FULL_IMAGE}"
                    }
                }
            }
        }
        
        stage('K8s Deploy') {
            steps {
                withKubeCredentials(kubectlCredentials: [[
                    caCertificate: '',
                    clusterName: 'test-cluster',
                    contextName: '',
                    credentialsId: 'K8s-token',
                    namespace: 'webapps',
                    serverUrl: 'https://B33AAF2FAFEBB286766B97890E4361FC.yl4.ap-south-1.eks.amazonaws.com'
                ]]) {
                    // Update deployment with new image
                    sh "kubectl set image deployment/bloggingapp-deployment bloggingapp=${FULL_IMAGE} -n webapps --record"

                    // Force imagePullPolicy=Always to avoid cached images
                    sh "kubectl patch deployment bloggingapp-deployment -n webapps -p '{\"spec\":{\"template\":{\"spec\":{\"containers\":[{\"name\":\"bloggingapp\",\"imagePullPolicy\":\"Always\"}]}}}}'"

                    // Rollout status check
                    sh "kubectl rollout status deployment/bloggingapp-deployment -n webapps"
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
                    namespace: 'webapps',
                    serverUrl: 'https://B33AAF2FAFEBB286766B97890E4361FC.yl4.ap-south-1.eks.amazonaws.com'
                ]]) {
                    sh "kubectl get pods -n webapps -o wide"
                    sh "kubectl get svc bloggingapp-ssvc -n webapps -o wide"
                }
            }
        }
    }
}
