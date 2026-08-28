pipeline {
    agent any
    environment {
        AWS_REGION = 'ap-south-2' 
        ECR_REPO = '361796581105.dkr.ecr.ap-south-2.amazonaws.com/daniel' 
        IMAGE_TAG = "v${BUILD_NUMBER}" 
        EKS_CLUSTER_NAME = 'daniel_cluster' 
        KUBECONFIG_PATH = '/opt/kube/config' 
        HELM_CHART_PATH = './Helm' 
    }
    stages {
        stage('SCM checkout') {
            steps {
                script {
                    git branch: 'main', 
                        credentialsId: 'git-cred', 
                        url: 'https://github.com/arulsamydaniel/K8s_Real_Project.git'
                }
            }
        }
        
        stage('SonarQube-SAST') {
            steps {
                script {
                    // Fetching the tool we named 'sonarscanner4' in Global Tools
                    def scannerHome = tool 'sonarscanner4'
                    
                    // Wrapping the execution in the environment configured as 'sonar-pro'
                    withSonarQubeEnv('sonar-pro') {
                        // Updated the projectKey to match our current repository name
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=nodejs-project"
                    }
                }
            }
        }
        stage('Build') {
            steps {
                script {
                    sh 'npm install'
                }
            }
        }
        stage('Docker Build Image') {
            steps {
                script {
                    // Build the Docker image with the dynamic version
                    sh "docker build -t my_pvt_repo:${IMAGE_TAG} ."
                    
                    // List Docker images to confirm the build
                    sh 'docker images'
                }
            }
        }
        stage('Docker Push to ECR') {
            steps {
                script {
                    def fullImageName = "${ECR_REPO}:${IMAGE_TAG}"
                    
                    // Dynamically extract just the registry URL (removes '/daniel')
                    def registryUrl = ECR_REPO.split('/')[0]
                    
                    // Login to ECR (AWS CLI automatically uses your Jenkins-EC2-Admin-Role!)
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${registryUrl}"
                    
                    // Tag the image for ECR
                    sh "docker tag my_pvt_repo:${IMAGE_TAG} ${fullImageName}"
                    
                    // Push the image to ECR
                    sh "docker push ${fullImageName}"
                }
            }
        }
        stage('Deploy on EKS') {
            steps {
                script {
                    // Set KUBECONFIG environment variable (pulls from your environment block)
                    env.KUBECONFIG = "${KUBECONFIG_PATH}"

                    // Update kubeconfig to interact with EKS cluster using your EC2 IAM role
                    sh "aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}"
                    
                    // Package the Helm chart (creates myrocketapp-0.1.0.tgz)
                    sh "helm package ${HELM_CHART_PATH}"
                    
                    // Deploy the Helm chart using the dynamic Jenkins WORKSPACE variable
                    sh """
                    helm upgrade --install myrocket ${WORKSPACE}/myrocketapp-0.1.0.tgz \
                    --set image.tag=${IMAGE_TAG}
                    """
                    
                    // Verify the deployment
                    sh 'helm ls'
                    sh 'kubectl get pods -o wide'
                    sh 'kubectl get svc'
                }
            }
        }
    }
}
