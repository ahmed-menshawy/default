def project = 'REPLACE_WITH_YOUR_PROJECT_ID'
def  appName = 'gceme'
def  feSvcName = "${appName}-frontend"
def  imageTag = "art4lab0.labs.mastercard.com:5001/${appName}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"

pipeline {
  agent {
    kubernetes {
      label 'ahmed-example'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
metadata:
labels:
  component: ci
spec:
  # Use service account that can deploy to all namespaces
  # serviceAccountName: cd-jenkins
  containers:
  - name: golang
    image: golang:1.10
    command:
    - cat
    tty: true
  - name: gcloud
    image: gcr.io/cloud-builders/gcloud:latest
    command:
    - cat
    tty: true
  - name: kubectl
    image: seldonio/k8s-deployer:latest
    command:
    - cat
    tty: true
  - name: docker
    image: docker:latest
    command:
    - cat
    tty: true
    volumeMounts:
    - mountPath: /var/run/docker.sock
      name: docker-sock

    env:
    - name: DOCKER_HOST
      value: "unix:///var/run/docker.sock"
    - name: DOCKER_OPTS
      value: "--insecure-registry=art4lab0.labs.mastercard.com:5001"
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/vcap/sys/run/docker/docker.sock
"""
}
  }
  stages {
    stage('Test') {
      steps {
        container('golang') {
          sh """
            ln -s `pwd` /go/src/sample-app
            cd /go/src/sample-app
            go test
          """
        }
      }
    }
    stage('Build and push image with Container Builder') {
      steps {
        container('docker') {
        withCredentials([usernamePassword(credentialsId: 'art4lab0-docker-deploy', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
          sh "docker build -t ${imageTag} ."
          sh "docker login art4lab0.labs.mastercard.com:5001 --username ${USER} --password ${PASS}"
          sh "docker push ${imageTag}"
          }
        }
      }
    }
    stage('Deploy Canary') {
      // Canary branch
      when { branch 'canary' }
      steps {
        container('kubectl') {
          // Change deployed image in canary to the one we just built
          sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/canary/*.yaml")
          sh("kubectl --namespace=production apply -f k8s/services/")
          sh("kubectl --namespace=production apply -f k8s/canary/")
          sh("echo http://`kubectl --namespace=production get service/${feSvcName} -o jsonpath='{.status.loadBalancer.ingress[0].ip}'` > ${feSvcName}")
        } 
      }
    }
    stage('Deploy Production') {
      // Production branch
      when { branch 'master' }
      steps{
        container('kubectl') {
        // Change deployed image in canary to the one we just built
          sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/production/*.yaml")
          sh("kubectl --namespace=production apply -f k8s/services/")
          sh("kubectl --namespace=production apply -f k8s/production/")
          sh("echo http://`kubectl --namespace=production get service/${feSvcName} -o jsonpath='{.status.loadBalancer.ingress[0].ip}'` > ${feSvcName}")
        }
      }
    }
    stage('Deploy Dev') {
      // Developer Branches
      when { 
        not { branch 'master' } 
        not { branch 'canary' }
      } 
      steps {
        container('kubectl') {
        withCredentials([file(credentialsId: 'kubeconfig-aiml-lower-env', variable: 'KUBECONFIG')]) {
          // Create namespace if it doesn't exist
          sh "kubectl --kubeconfig ${KUBECONFIG} cluster-info"
          echo "${KUBECONFIG}"
          sh("kubectl --kubeconfig ${KUBECONFIG} --cluster=exp get ns ${env.BRANCH_NAME} || kubectl --kubeconfig ${KUBECONFIG} --cluster=exp create ns ${env.BRANCH_NAME}")
          // Don't use public load balancing for development branches
          sh("sed -i.bak 's#LoadBalancer#ClusterIP#' ./k8s/services/frontend.yaml")
          sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/dev/*.yaml")
          sh("kubectl --kubeconfig ${KUBECONFIG} --namespace=${env.BRANCH_NAME} --cluster=exp apply -f k8s/services/")
          sh("kubectl --kubeconfig ${KUBECONFIG} --namespace=${env.BRANCH_NAME} --cluster=exp apply -f k8s/dev/")
          echo 'To access your environment run `kubectl proxy`'
          echo "Then access your service via http://localhost:8001/api/v1/proxy/namespaces/${env.BRANCH_NAME}/services/${feSvcName}:80/"
          }
        }
      }     
    }
  }
}
