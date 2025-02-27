buildPlugin(
  useContainerAgent: true,
  configurations: [
    [platform: 'linux', jdk: 11],
    [platform: 'windows', jdk: 11],
])
pipeline {
    agent any
    environment {
        AWS_REGION = 'us-east-1' 
    }
    environment {
        DOCKER_IMAGE_NAME = "$DOCKER_REG_URL/docker-local/hello-frog:1.0.0"
    }
    stages {
         stage('Clone') {
            steps {
                git branch: 'master', url: "https://github.com/jfrog/project-examples.git"
            }
        }

        stage('Build Docker image') {
            steps {
                script {
                    docker.build("$DOCKER_IMAGE_NAME", 'docker-oci-examples/docker-example')
                }
            }
        }stage('Scan and push image') {
            steps {
                dir('docker-oci-examples/docker-example/') {
                    // Scan Docker image for vulnerabilities
                    jf 'docker scan $DOCKER_IMAGE_NAME'

                    // Push image to Artifactory
                    jf 'docker push $DOCKER_IMAGE_NAME'
                }
            }
        }
        stage('Set AWS Credentials') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws' 
                ]]) {
                    sh '''
                    echo "AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID"
                    aws sts get-caller-identity
                    '''
                }
            }
        }
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/Rbilli51614/jenkins-jfrog-plugin.git' 
            }
        }
        stage('Initialize Terraform') {
            steps {
                sh '''
                terraform init
                '''
            }
        }
        stage('Plan Terraform') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws'
                ]]) {
                    sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    terraform plan -out=tfplan
                    '''
                }
            }
        }
        stage('Apply Terraform') {
            steps {
                input message: "Approve Terraform Apply?", ok: "Deploy"
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws'
                ]]) {
                    sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    terraform apply -auto-approve tfplan
                    '''
                }
            }
        }
    }
    post {
        success {
            echo 'Terraform deployment completed successfully!'
        }
        failure {
            echo 'Terraform deployment failed!'
        }
    }
}