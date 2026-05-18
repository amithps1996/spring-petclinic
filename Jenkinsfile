pipeline {

  agent any

  environment {
    GITHUB_REPO   = 'https://github.com/amithps1996/spring-petclinic.git'
    JFROG_URL     = 'http://10.0.0.4:8082/artifactory'
    JFROG_REPO    = 'libs-snapshot-local'
    DOCKER_IMAGE  = 'amithps1996/spring-petclinic'
    DOCKER_TAG    = 'latest'
    K8S_NAMESPACE = 'jenkins-builds'
  }

  stages {

    // ─────────────────────────────────────────
    // STAGE 1: Checkout — runs on default agent
    // ─────────────────────────────────────────
    stage('Checkout') {
      steps {
        echo '=== Stage 1: Checkout Source Code ==='
        git branch: 'main',
            credentialsId: 'github-creds',
            url: "${GITHUB_REPO}"
        sh 'ls -la k8s/'
      }
    }

    // ─────────────────────────────────────────
    // STAGE 2: Build & Test
    // Docker image used as agent for this stage
    // ─────────────────────────────────────────
    stage('Build and Test') {
      agent {
        docker {
          image 'maven:3.9-eclipse-temurin-17'
          reuseNode true
          args '-v $HOME/.m2:/root/.m2'
        }
      }
      steps {
        echo '=== Stage 2: Maven Build & Test (Docker image as agent) ==='
        sh 'mvn clean package -DskipTests=false -Dcheckstyle.skip=true --batch-mode'
        sh 'ls -la target/spring-petclinic-*.jar'
      }
      post {
        always {
          junit allowEmptyResults: true,
                testResults: '**/target/surefire-reports/*.xml'
        }
      }
    }

    // ─────────────────────────────────────────
    // STAGE 3: Push Artifact to JFrog
    // ─────────────────────────────────────────
    stage('Push to JFrog') {
      steps {
        echo '=== Stage 3: Push Artifact to JFrog Artifactory ==='
        rtServer(
          id:            'ART_SERVER',
          url:           "${JFROG_URL}",
          credentialsId: 'jfrog-creds'
        )
        rtUpload(
          serverId: 'ART_SERVER',
          spec: """{
            "files": [{
              "pattern": "target/spring-petclinic-*.jar",
              "target": "${JFROG_REPO}/spring-petclinic/"
            }]
          }"""
        )
        rtPublishBuildInfo(serverId: 'ART_SERVER')
        echo '=== Artifact Uploaded to JFrog ==='
      }
    }

    // ─────────────────────────────────────────
    // STAGE 4: Build & Push Docker Image
    // Docker container as Jenkins SLAVE
    // ─────────────────────────────────────────
    stage('Build and Push Docker Image') {
      steps {
        echo '=== Stage 4: Build & Push Docker Image (Docker slave) ==='
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
            def img = docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
            img.push()
            img.push('latest')
          }
        }
        echo '=== Docker Image Pushed to Docker Hub ==='
      }
    }

    // ─────────────────────────────────────────
    // STAGE 5: Deploy to AKS
    // K8s plugin — pod with jnlp + kubectl
    // ─────────────────────────────────────────
    stage('Deploy to AKS') {
      agent {
        kubernetes {
          cloud 'aks-cloud'
          yaml """
apiVersion: v1
kind: Pod
metadata:
  namespace: jenkins-builds
  labels:
    app: petclinic-deploy
spec:
  serviceAccountName: jenkins-sa
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest
  - name: kubectl
    image: alpine/k8s:1.28.2
    command:
    - cat
    tty: true
"""
        }
      }
      steps {
        echo '=== Stage 5: Deploy to AKS ==='
        // Checkout k8s manifests inside the pod
        git branch: 'main',
            credentialsId: 'github-creds',
            url: "${GITHUB_REPO}"
        container('kubectl') {
          sh '''
            echo "=== Applying K8s Manifests ==="
            kubectl apply -f k8s/deployment.yaml
            kubectl apply -f k8s/service.yaml

            echo "=== Waiting for Rollout ==="
            kubectl rollout status deployment/petclinic-deployment \
              -n jenkins-builds --timeout=300s

            echo "=== Live Pods ==="
            kubectl get pods -n jenkins-builds

            echo "=== Services ==="
            kubectl get svc -n jenkins-builds
          '''
        }
      }
    }

    // ─────────────────────────────────────────
    // STAGE 6: Manual Approval Gate
    // ─────────────────────────────────────────
    stage('Manual Approval') {
      agent none
      steps {
        echo '=== Stage 6: Awaiting Manual Approval ==='
        timeout(time: 10, unit: 'MINUTES') {
          input(
            message: 'Deployment is live on AKS. Approve to destroy?',
            ok: 'Approve & Destroy'
          )
        }
      }
    }

    // ─────────────────────────────────────────
    // STAGE 7: Destroy Deployment
    // After confirmation, tear down the pod
    // ─────────────────────────────────────────
    stage('Destroy Deployment') {
      agent {
        kubernetes {
          cloud 'aks-cloud'
          yaml """
apiVersion: v1
kind: Pod
metadata:
  namespace: jenkins-builds
spec:
  serviceAccountName: jenkins-sa
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest
  - name: kubectl
    image: alpine/k8s:1.28.2
    command:
    - cat
    tty: true
"""
        }
      }
      steps {
        echo '=== Stage 7: Destroying Deployment After Approval ==='
        container('kubectl') {
          sh '''
            kubectl delete deployment petclinic-deployment \
              -n jenkins-builds --ignore-not-found=true

            kubectl delete service petclinic-service \
              -n jenkins-builds --ignore-not-found=true

            echo "=== Remaining Resources ==="
            kubectl get pods,deployments,services -n jenkins-builds
          '''
        }
      }
    }

  }  // end stages

  // ─────────────────────────────────────────
  // POST ACTIONS
  // ─────────────────────────────────────────
  post {
    always {
      echo '=== Post: Cleaning Workspace ==='
      cleanWs()
    }
    success {
      echo '=== Post: PIPELINE SUCCESS ==='
    }
    failure {
      echo '=== Post: PIPELINE FAILURE ==='
    }
    unstable {
      echo '=== Post: PIPELINE UNSTABLE ==='
    }
    aborted {
      echo '=== Post: PIPELINE ABORTED ==='
    }
  }

}
