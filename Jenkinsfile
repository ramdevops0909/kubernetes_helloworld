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
                input 'Deploy to Production?'
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'helloworld-k8s-service.yml',
                    enableConfigSubstitution: true
                )
            }
        }
  }
}
