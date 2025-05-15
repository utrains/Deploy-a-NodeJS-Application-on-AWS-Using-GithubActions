pipeline {
    agent any

    stages {
        stage ('Create the infrastructure on AWS'){
            steps {
                sh '''
                cd ecr
                terraform init
                terraform apply -auto-approve
                '''
            }
        }

        stage ('Build backend and frontend images to ECR'){
            steps {
                sh '''
                cd ecr
                ./docker-build-script.sh
                '''
            }
        }

        stage ('Push backend and frontend images to ECR'){
            steps {
                sh '''
                cd ecr
                ./docker-push-script.sh
                '''
            }
        }
        
        stage ('Initialising the terraform code to Launch the frontend and the backend app'){
            steps{
                
                sh 'terraform init'
            }
        }

        stage ('Deploying the app to ECS'){
            steps{
                sh 'terraform apply --auto-approve'
            }
        }
    }
}