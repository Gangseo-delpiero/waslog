pipeline {
    agent any
    environment {
        IAM_ROLE = 'lee'        
        ROLE_ACCOUNT = credentials('roleAccount')
        REGION = credentials('AWS_REGION')
        ECR_URI = credentials('ECR_URI')
        ECR_REPO='hsw_was'
        TAG = "1.$BUILD_NUMBER"
    }
 
    stages {          
        stage('Maven Build') {
            steps {
                script {
                    sh "./mvnw dependency:resolve"
                    sh "./mvnw -DskipTests=true package"
                }
            }
        } 

        stage('Docker Image WAS Build') {
            steps {
                sh 'docker build --no-cache -t was:latest -f Dockerfile.was .'
            }
            post {
                failure {
                    echo 'Docker image WAS build failure !'
                }
                success {
                    echo 'Docker image WAS build success !'
                }
            }
        }

        stage('WAS test run') {
            steps {
                sh 'docker run -d -p 8000:8000 --rm --name was was:latest'
            }
            post {
                failure {
                    sh 'docker stop was'
                }
            }
        }
        
        stage('was cleanup test') {
            steps {
                sh 'docker stop was'
            }
        } 
        
        stage('push WAS to ECR') {
            steps {
                script {     
                    withEnv(['ROLE_ACCOUNT=$ROLE_ACCOUNT','REGION=$REGION']){
                        withAWS(role: IAM_ROLE, externalId: 'externalId'){
                            sh 'aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin $ECR_URI'
                            sh 'docker tag was:latest $ECR_URI/$ECR_REPO:$TAG'
                            sh 'docker push $ECR_URI/$ECR_REPO:$TAG'
                        }      
                    } 
                    
                }
            }
        }
 
        stage('docker image remove') {
            steps {
                sh 'docker rmi -f $(docker images -aq)'
            }
        }
          
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: '36f16a8a-689a-4c67-af59-80357914cd9e', url: 'https://github.com/SeoungWan/argocd.git'
            } 
        }
    
        stage('K8S Manifest Update') {
            steps {
                sshagent(credentials: ['35c59d94-6fb0-4f12-9bd0-da81814c0eb8']) {
                    sh """
                        #!/usr/bin/env bash
                        set +x
                        export GIT_SSH_COMMAND="ssh -oStrictHostKeyChecking=no"
                        git config --global user.email "qak8883@naver.com"
                        git checkout main
                        kustomize edit set image $ECR_URI/$ECR_REPO:$TAG
                        git commit -a -m "updated the image tag:$TAG"
                        git remote set-url origin git@github.com:SeoungWan/argocd.git
                        git push origin main
                    """
                }
            }
            post {
                failure {
                    echo 'K8S Manifest Update failure !'
                }
                success {
                    echo 'K8S Manifest Update success !'
                }
            }
        }    
    }
    // post  {
    //     success {
    //         slackSend (
    //             channel: '#jenkins',
    //             color: '#00ff00',
    //             message: "SUCCESSFUL: WAS '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
    //         )
    //     }
    //     failure {
    //         slackSend (
    //             channel: '#jenkins',
    //             color: '#ff0000',
    //             message: "FAILED: WAS '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
    //         )
    //     }
    // }
}