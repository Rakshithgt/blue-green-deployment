pipeline {
    agent any

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')
    }

    environment {
        IMAGE_NAME = "rakshithgt96/bankapp"
        TAG = "${params.DOCKER_TAG}"
        KUBE_NAMESPACE = 'webapps'
        SCANNER_HOME = tool 'sonar-scanner'
    }

    tools {
        maven 'maven3'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-token', url: 'https://github.com/Rakshithgt/blue-green-deployment.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh "trivy fs --format table -o fs.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=nodejsmysql -Dsonar.projectName=nodejsmysql -Dsonar.sources=. -Dsonar.java.binaries=target/classes"
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
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
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push ${IMAGE_NAME}:${TAG}"
                    }
                }
            }
        }

        stage('Configure EKS Kubeconfig') {
           steps {
             withCredentials([usernamePassword(credentialsId: 'aws-cred', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                 withEnv(["AWS_DEFAULT_REGION=ap-south-1"]) {
                    sh '''
                       echo "AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID"
                       echo "AWS_SECRET_ACCESS_KEY: $AWS_SECRET_ACCESS_KEY"
                       aws eks update-kubeconfig --region ap-south-1 --name gtr-cluster
                       kubectl get nodes
                    '''
                 }
              }
          }
       }


        stage('Deploy MySQL Deployment and Service') {
            steps {
                sh """
                   kubectl get nodes
                   kubectl apply -f mysql-ds.yml -n ${KUBE_NAMESPACE}
                """
            }
        }

        stage('Deploy SVC-APP') {
            steps {
                sh """
                    if ! kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}; then
                        kubectl apply -f bankapp-service.yml -n ${KUBE_NAMESPACE}
                    fi
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    def deploymentFile = params.DEPLOY_ENV == 'blue' ? 'app-deployment-blue.yml' : 'app-deployment-green.yml'
                    sh "kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE}"
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
                    sh """
                        kubectl patch service bankapp-service -p '{"spec": {"selector": {"app": "bankapp", "version": "${newEnv}"}}}' -n ${KUBE_NAMESPACE}
                    """
                    echo "✅ Traffic has been switched to the '${newEnv}' environment."
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    def verifyEnv = params.DEPLOY_ENV
                    sh """
                        kubectl get pods -l version=${verifyEnv} -n ${KUBE_NAMESPACE}
                        kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}
                    """
                }
            }
        }
    }
}
