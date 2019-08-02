# Kubernetes Deployment Using Jenkins Pipeline

This repo contains code for deploying the Spring boot Application(Helloworld.jar) in Kubernetes Google cloud cluster environment.
Using Jenkins Pipe line, I have performed following things
      	1) Created Global Credentials (dockerhub,Jenkins,kubeconfig) in Jenkins console
	2) Created the Dockerfile and dockerized the helloworld.jar file
        3) Build the image and pushed the image into Docker Hub registry and removed unused docker  images
        4) Prepared the Kubernetes deployment,service yml files (Including liveness and readines probes) for creation of K8S Infrastructure using Jenkins Pipeline

Dockerfile:
----------
   Using Dockerfile, I am able to dockerize the helloworld microservice

	```
	FROM openjdk:8-jdk-alpine
	MAINTAINER RamaGopal <ram.devops0909@gmail.com>
	EXPOSE 8080
	COPY helloworld.jar app.jar
	ENTRYPOINT ["java", "-Djava.security.egd=file:/dev/./urandom", "-jar", "app.jar"]

	```
helloworld-k8s-service.yml:
---------------------------
Below is the helloworld kubernetes service yml file. Using following deployment.yaml file we can spinup 3 pods. I successfully deployed (rolling update on Kubernetes cluser ( its has 3 nodes ( kubemaster,kubenode-1,kubenode-2)) above dockerized microservice into google cloud cluster environment.

Using service we can access Helloworld application using loadbancer Ip address with port "8080"

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-deployment
spec:
  replicas: 3
  minReadySeconds: 30
  selector:
    matchLabels:
      app: hello-world-k8s
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: hello-world-k8s
    spec:
      containers:
      - name: hello-world-k8s
        image: $registry:$BUILD_NUMBER
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 10
          timeoutSeconds: 1
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 10
          timeoutSeconds: 1
---
kind: Service
apiVersion: v1
metadata:
  name:  hello-world-service
spec:
  selector:
    app:  hello-world-k8s
  type:  LoadBalancer
  ports:
  - name:  http
    port:  8080
    protocol: TCP

```
```
Even I tried to access the Application using Node port as well. In order to access the application using NodePort from out side the Kubernetes cluster, I have enabeld the the firewall for the perticular port using below command

C:\Program Files (x86)\Google\Cloud SDK>gcloud compute firewall-rules create my-rule-k8s --allow=tcp:30005
Creating firewall...\Created [https://www.googleapis.com/compute/v1/projects/cohesive-ridge-226016/global/firewalls/my-rule-k8s].
Creating firewall...done.
NAME         NETWORK  DIRECTION  PRIORITY  ALLOW      DENY  DISABLED
my-rule-k8s  default  INGRESS    1000      tcp:30005        False

Once enabled port (30005) ,I was able to access the application using this URL (http://Node_public_ip:30005)

kind: Service
apiVersion: v1
metadata:
  name:  hello-world-service
spec:
  selector:
    app:  hello-world-k8s
  type:  NodePort
  ports:
  - name:  http
    port:  8080
    nodePort: 30005
    protocol: TCP

```
```
Jenkinsfile:
------------
Following Jenkins file would clone the Dockerfile and helloworld.jar file form the git repository and then dockerize (creat the image) and later it will tag and then push to docker hub repository.
Later this jenkins file would call above helloworld-k8s-service.yml file. This yml file will create pods and service.
Using loadbalancer we can access deployed application (http://<LoadbalancerIp>:port)


pipeline {
  environment {
    registry = "ramdevops0909/kubernetes_helloworld"
    registryCredential = 'dockerhub'
    dockerImage = ''
  }
  agent any
  stages {
    stage('Cloning Git') {
      steps {
        git 'https://github.com/ramdevops0909/kubernetes_helloworld.git'
      }
    }
    stage('Building image') {
      steps{
        script {
          dockerImage = docker.build registry + ":$BUILD_NUMBER"
        }
      }
    }
    stage('Push the docker Image') {
      steps{
        script {
          docker.withRegistry( '', registryCredential ) {
            dockerImage.push()
          }
        }
      }
    }
    stage('Remove Unused docker image') {
      steps{
        sh "docker rmi $registry:$BUILD_NUMBER"
      }
    }
    stage('DeployToK8SCluster') {
            steps {
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'helloworld-k8s-service.yml',
                    enableConfigSubstitution: true
                )
            }
        }
  }
}

```

In order to access above deployed helloworld micro service, we need to use this URL `http://<Loadbalancer>:<<port>>` in your browser, you should be able to see as below
	Hello World from upday!
