# Using Jenkins to deploy Spring Boot Applications into EKS

### AWS Infrastructure

![Infrastructure](.files/EKS%20Veeva-AWS%20Infra.drawio.png)

### Jenkins Pipeline

![Infrastructure](.files/EKS%20Veeva-Jenkins.drawio.png)

Link: http://ec2-3-144-94-235.us-east-2.compute.amazonaws.com:8080/

# Used plugins

- Docker
- Docker Pipeline
- Kubernetes
- Kubernetes Credentials
- Kubernetes Client API
- GitHub Integration

# Pipeline scripts

CI
```groovy
pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        registry = "092369361076.dkr.ecr.us-east-1.amazonaws.com/test-ecr"
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
                    sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 092369361076.dkr.ecr.us-east-1.amazonaws.com'
                    sh 'docker push 092369361076.dkr.ecr.us-east-1.amazonaws.com/test-ecr:latest'
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
                    withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'Kubeconfig-EKS-us-east-2', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                        script {
                            def exists = sh(
                                    script: "kubectl get deployment springboot-app",
                                    returnStatus: true
                            ) == 0
                            if (exists) {
                                echo "Delete Deployment and Service"
                                sh ('kubectl delete -f  eks-deploy-k8s.yaml')
                                sh ('kubectl apply -f  eks-deploy-k8s.yaml')
                            } else {
                                echo "Create Deployment and Service"
                                sh ('kubectl apply -f  eks-deploy-k8s.yaml')
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

# Update EKS to deploy Application Load Balancer instead of Classic Load Balancer

First, you need to install the Kubernetes Service controller for Elastic Load Balancing v2 by running the following command:

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/v2_1_3_full.yaml

```

This will install the AWS Load Balancer Controller in your cluster.

After the controller is installed, you can create a Kubernetes Service object of type LoadBalancer and add an annotation to it specifying that it should use an Application Load Balancer.

```yaml
apiVersion: v1
kind: Service
metadata:
    name: my-service
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
    type: LoadBalancer
    selector:
      app: my-app
    ports:
    - name: http
      port: 80
      targetPort: 8080
```

The annotation `service.beta.kubernetes.io/aws-load-balancer-type: "nlb"` specifies that an Application Load Balancer should be used instead of a Classic Load Balancer.

Lastly, apply the YAML file to create the Service object.

```shell
kubectl apply -f my-service.yaml
```
