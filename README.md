# Using Jenkins to deploy Spring Boot Applications into EKS

### AWS Infrastructure

![Infrastructure](.files/EKS%20Veeva-AWS%20Infra.drawio.png)

### Jenkins Pipeline

![Infrastructure](.files/EKS%20Veeva-Jenkins.drawio.png)

Link: http://ec2-3-239-79-165.compute-1.amazonaws.com:8080/

# Used plugins

- Docker
- Docker Pipeline
- Kubernetes
- Kubernetes Credentials
- Kubernetes Client API
- GitHub Integration

# Pipeline scripts

NOTE: Update the AWS account ID.

CI
```groovy
pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        registry = "012345678901.dkr.ecr.us-east-1.amazonaws.com/test-ecr"
    }

    stages {
        stage('Checkout the Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '', url: 'https://github.com/ricardorqr/springboot-app']]])
            }
        }

        stage('Build the JAR') {
            steps {
                sh 'mvn clean install'
            }
        }

        stage('Building the Image') {
            steps {
                script {
                    dockerImage = docker.build registry
                }
            }
        }

        stage('Push into the ECR') {
            steps{
                script {
                    sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 012345678901.dkr.ecr.us-east-1.amazonaws.com'
                    sh 'docker push 012345678901.dkr.ecr.us-east-1.amazonaws.com/test-ecr:latest'
                }
            }
        }
    }
}
```

CD
```groovy
pipeline {
    agent any

    stages {
        stage('Checkout the Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '', url: 'https://github.com/ricardorqr/springboot-app']]])
            }
        }

        stage('Deploy into EKS') {
            steps{
                script {
                    kubeconfig(credentialsId: 'Kubeconfig', serverUrl: '') {
                        script {
                            def exists = sh(
                                    script: "kubectl get deployment springboot-app",
                                    returnStatus: true
                            ) == 0
                            if (exists) {
                                echo "Update deployment image"
                                sh ('kubectl rollout restart deployment/springboot-app')
                                sh ('kubectl rollout status deployment/springboot-app')
                            } else {
                                echo "Create deployment and service manifest"
                                sh ('kubectl apply -f eks-deploy-k8s.yaml')
                            }
                        }
                    }
                }
            }
        }
    }
}
```

### Kubernetes Infrastructure

![Infrastructure](.files/EKS%20Veeva-Kubernetes.drawio.png)