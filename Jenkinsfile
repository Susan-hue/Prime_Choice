pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'DESTROY_INFRA',
            defaultValue: false,
            description: '⚠️ Destroy ALL Terraform-managed infrastructure (EKS, VPC, etc.)'
        )
    }

    environment {
        AWS_REGION = 'us-east-1'
        DOCKERHUB_USERNAME = 'susan22283'

        IMAGE_NAME = 'prime-choice-app'
        EKS_CLUSTER_NAME = 'my-cluster'
        TAG = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Verify AWS Identity') {
            when { expression { params.DESTROY_INFRA == false } }
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-credentials']
                ]) {
                    sh '''
                      export AWS_DEFAULT_REGION=$AWS_REGION
                      aws --version
                      aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('Terraform Init & Apply (EKS)') {
            when { expression { params.DESTROY_INFRA == false } }
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-credentials']
                ]) {
                    sh '''
                      export AWS_DEFAULT_REGION=$AWS_REGION
                      terraform init
                      terraform validate
                      terraform apply -auto-approve
                    '''
                }
            }
        }

        stage('Configure AWS & EKS') {
            when { expression { params.DESTROY_INFRA == false } }
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-credentials']
                ]) {
                    sh '''
                        export AWS_DEFAULT_REGION=$AWS_REGION

                        aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME

                        kubectl cluster-info
                        kubectl get nodes
                    '''
                }
            }
        }

        stage('Docker Login') {
            when { expression { params.DESTROY_INFRA == false } }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                }
            }
        }

        stage('Build Docker Image') {
            when { expression { params.DESTROY_INFRA == false } }
            steps {
                sh "docker build -t $DOCKERHUB_USERNAME/$IMAGE_NAME:$TAG ."
            }
        }

        stage('Push Docker Image') {
            when { expression { params.DESTROY_INFRA == false } }
            steps {
                sh "docker push $DOCKERHUB_USERNAME/$IMAGE_NAME:$TAG"
            }
        }

        stage('Update YAMLs & Deploy to EKS') {
            when { expression { params.DESTROY_INFRA == false } }
            steps {
                dir('kubernetes') {
                    sh """
                      echo "Updating Django image tag..."
                      sed -i "s|image: .*|image: $DOCKERHUB_USERNAME/$IMAGE_NAME:$TAG|g" django-deployment.yaml

                      echo "Applying Kubernetes manifests..."
                      kubectl apply -f namespace.yaml

                      kubectl apply -f postgres-secret.yaml
                      kubectl apply -f django-secret.yaml
                      kubectl apply -f django-configmap.yaml

                      kubectl apply -f postgres-pvc.yaml

                      kubectl apply -f postgres-deployment.yaml
                      kubectl apply -f postgres-service.yaml

                      kubectl apply -f django-deployment.yaml
                      kubectl apply -f django-service.yaml

                      echo "Waiting for deployments to be ready..."
                      kubectl rollout status deployment/postgres -n django-app --timeout=300s
                      kubectl rollout status deployment/django -n django-app --timeout=300s
                    """
                }
            }
        }

        stage('Terraform Destroy (EKS & Infra)') {
            when { expression { params.DESTROY_INFRA == true } }
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-credentials']
                ]) {
                    sh '''
                      export AWS_DEFAULT_REGION=$AWS_REGION
                      terraform init
                      terraform destroy -auto-approve
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
        failure {
            echo '❌ Pipeline failed!'
        }
        success {
            echo '✅ Pipeline completed successfully!'
        }
    }
}
