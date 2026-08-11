pipeline {
    agent any
    
    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')
    }
    tools {
        maven 'maven3'
        jdk 'jdk17'
    }
    
    environment {
        IMAGE_NAME = "bjrules/bankapp"
        TAG = "${params.DOCKER_TAG}"  // The image tag now comes from the parameter
        KUBE_NAMESPACE = 'webapps'
        SCANNER_HOME = tool 'sonar-scanner'
       // JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-amd64'
        // PATH = "${JAVA_HOME}/bin:${env.PATH}"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Bjrules/Blue-Green-Deployment-II.git'
            }
        }
        stage('Compile'){
            steps {
                sh "mvn compile"
            }
        }

        stage('Testing'){
            steps {
                sh "mvn test -DskipTests=true"
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs --format table -o filescan.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=GCBANK -Dsonar.projectName=GCBANK -Dsonar.java.binaries=target"
                }
            }
        }
        stage('Quality gates Check'){
            steps {
              timeout(time: 1, unit: 'HOURS') {
                waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }

        
         stage('Publish To Nexus') {
            steps {
               // withMaven(globalMavenSettingsConfig: 'BJ-settings', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                        sh "mvn deploy -DskipTests=true"
                nexusArtifactUploader credentialsId: 'Nexus-server', groupId: 'nil', nexusUrl: '98.93.227.199:8081', nexusVersion: 'nexus3', protocol: 'http', repository: 'maven-snapshots', version: '0.0.1'
                   // }
            }
        }
        stage('Docker build & Tag Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'Dockerhub-cred') {
                        sh "docker build -t ${IMAGE_NAME}:${TAG} ."
                    }
                }
            }
        }
        
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --format table -o image.html ${IMAGE_NAME}:${TAG}"
            }
        }
        
        stage('Docker Push Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'Dockerhub-cred') {
                        sh "docker push ${IMAGE_NAME}:${TAG}"
                    }
                }
            }
        }
        
        stage('Deploy MySQL Deployment and Service') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'bnj-cluster', contextName: '', credentialsId: 'K8s-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://9AC89BB362F279E822E7111ECD33540F.gr7.us-east-1.eks.amazonaws.com') {
                        sh "kubectl apply -f mysql-ds.yml -n ${KUBE_NAMESPACE}"  // Ensure you have the MySQL deployment YAML ready
                    }
                }
            }
        }
        
        stage('Deploy SVC-APP') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'bnj-cluster', contextName: '', credentialsId: 'K8s-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://9AC89BB362F279E822E7111ECD33540F.gr7.us-east-1.eks.amazonaws.com') {
                        sh """ if ! kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}; then
                                kubectl apply -f bankapp-service.yml -n ${KUBE_NAMESPACE}
                              fi
                        """
                   }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    def deploymentFile = ""
                    if (params.DEPLOY_ENV == 'blue') {
                        deploymentFile = 'app-deployment-blue.yml'
                    } else {
                        deploymentFile = 'app-deployment-green.yml'
                    }

                    withKubeConfig(caCertificate: '', clusterName: 'bnj-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://9AC89BB362F279E822E7111ECD33540F.gr7.us-east-1.eks.amazonaws.com') {
                        sh "kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE}"
                    }
                }
            }
        }
        
        stage('Switch Traffic Between Blue & Green Environment') {
            when {
                expression { return params.SWITCH_TRAFFIC }
            }
            steps {
                script {
                    def newEnv = params.DEPLOY_ENV

                    // Always switch traffic based on DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'bnj-cluster', contextName: '', credentialsId: 'K8s-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://9AC89BB362F279E822E7111ECD33540F.gr7.us-east-1.eks.amazonaws.com') {
                        sh '''
                            kubectl patch service bankapp-service -p "{\\"spec\\": {\\"selector\\": {\\"app\\": \\"bankapp\\", \\"version\\": \\"''' + newEnv + '''\\"}}}" -n ${KUBE_NAMESPACE}
                        '''
                    }
                    echo "Traffic has been switched to the ${newEnv} environment."
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    def verifyEnv = params.DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'bnj-cluster', contextName: '', credentialsId: 'K8s-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://9AC89BB362F279E822E7111ECD33540F.gr7.us-east-1.eks.amazonaws.com') {
                        sh """
                        kubectl get pods -l version=${verifyEnv} -n ${KUBE_NAMESPACE}
                        kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}
                        """
                    }
                }
            }
        }
    }
}
