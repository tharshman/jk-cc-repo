pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = "us-east-1"
        AWS_CREDENTIALS = credentials('jk-aws-credentials')
        ECR_REPO = "976193221400.dkr.ecr.us-east-1.amazonaws.com/jk-webapp"
        ECS_CLUSTER = "jk-webapp-cluster"
        ECS_SERVICE = "jk-webapp-td-service-flvo6c8w"
    }

    stages {
        stage('Checkout Source from GitHub') {
            steps {
                git branch: 'main', credentialsId: 'jk-gh-tk', url: 'https://github.com/tharshman/jk-cc-repo.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build --no-cache -t $ECR_REPO:$BUILD_NUMBER .
                '''
            }
        }

        stage('Login to AWS ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_REPO
                '''
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                sh '''
                    docker push $ECR_REPO:$BUILD_NUMBER
                '''
            }
        }

        stage('Register ECS Task Definition and Update Service') {
            steps {
                script {
                    def registerOutput = sh(
                        script: """
                            aws ecs register-task-definition \
                                --family jk-webapp-td \
                                --execution-role-arn arn:aws:iam::976193221400:role/ecsTaskExecutionRole \
                                --network-mode bridge \
                                --requires-compatibilities EC2 \
                                --cpu "1024" \
                                --memory "512" \
                                --container-definitions '[
                                  {
                                    "name": "jk-webapp-ctr",
                                    "image": "${ECR_REPO}:${BUILD_NUMBER}",
                                    "essential": true,
                                    "memory": 512,
                                    "portMappings": [
                                      {
                                        "containerPort": 3000,
                                        "hostPort": 3000
                                      }
                                    ]
                                  }
                                ]'
                        """,
                        returnStdout: true
                    ).trim()

                    // Extract task definition ARN
                    def taskDefArn = readJSON(text: registerOutput).taskDefinition.taskDefinitionArn

                    // Use that ARN in the ECS service update
                    sh """
                        aws ecs update-service \
                            --cluster $ECS_CLUSTER \
                            --service $ECS_SERVICE \
                            --task-definition $taskDefArn \
                            --force-new-deployment
                    """
                }
            }
        }
    }
