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
                    // Replace the URL with your new DevOps project repo once you have it
                    git branch: 'main', 
                        credentialsId: 'git-cred', 
                        url: 'https://github.com/arulsamydaniel/K8s_Real_Project.git'
                }
            }
        }
        stage('SonarQube-SAST') {
            steps {
                script {
                    def scannerHome = tool 'sonarscanner4'
                    withSonarQubeEnv('sonar-pro') {
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
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials-vgs' // Your Jenkins AWS credentials ID
                    ]]) {
                        def fullImageName = "${ECR_REPO}:${IMAGE_TAG}"
                        
                        // Login to ECR
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}"
                        
                        // Tag the image for ECR
                        sh "docker tag my_pvt_repo:${IMAGE_TAG} ${fullImageName}"
                        
                        // Push the image to ECR
                        sh "docker push ${fullImageName}"
                    }
                }
            }
        }
        stage('Deploy on EKS') {
            steps {
                script {
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials-vgs' // Replace with your Jenkins AWS credentials ID
                    ]]) {
                        
                        // Set KUBECONFIG environment variable
                        env.KUBECONFIG = '/opt/kube/config'

                        // Update kubeconfig to interact with EKS cluster
                        sh "aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}"
                        
                        // Create Docker registry secret for Helm
                        sh """
                        kubectl create secret generic helm \
                            --from-file=.dockerconfigjson=/opt/docker/config.json \
                            --type=kubernetes.io/dockerconfigjson \
                            --dry-run=client -o yaml > secret.yaml
                        kubectl apply -f secret.yaml 
                        """
                        
                        // Package and deploy the Helm chart
                        sh """
                        helm package ${HELM_CHART_PATH}
                        pwd
                        #helm install myrocket /var/lib/jenkins/workspace/project/myrocketapp-0.1.0.tgz --kubeconfig /opt/kube/config
                        helm upgrade --install myrocket /var/lib/jenkins/workspace/rocket/myrocketapp-0.1.0.tgz \
                        --set image.tag=v${BUILD_NUMBER} 
                        """
                        
                        // List Helm releases and Kubernetes resources
                        sh 'helm ls'
                        sh 'kubectl get pods -o wide'
                        sh 'kubectl get svc'
                    }
                }
            }
        }
    }
}
