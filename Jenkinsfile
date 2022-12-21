pipeline {
    agent any


  environment {
    // Adjust variables below
    ARGOCD_SERVER     = "10.13.2.10:30443"
    APP_MANIFEST_REPO = "https://github.com/yolaadipratama/simple-webapp-manifest.git"
    IMAGE_NAME        = "docker.io/yolaadipratama/webapp"


    // Do not edit variables below
    ARGOCD_OPTS           = "--grpc-web --insecure"
    TAG                   = sh (script: "date +%y%m%d%H%M", returnStdout: true).trim()
    APP_MANIFEST_PATH     = sh (script: "echo $APP_MANIFEST_REPO | sed 's/.*github.com//g'", returnStdout: true).trim()
    DOCKERHUB_CREDENTIALS = credentials('dockerhub-cred')
  }

  stages {
    stage('Preparation') {
      steps {
        sh "echo App Version = $TAG"
      }
    }

    stage('Test') {
      steps {
        sh "html5validator html/index.html"
      }
      post {
        success {
           echo "Test Successful"
        }
        failure {
           echo "Test Failed"
        }
      }
    }

    stage("Build & Push Image"){
      steps{
        sh """
        docker login -u ${DOCKERHUB_CREDENTIALS_USR} -p ${DOCKERHUB_CREDENTIALS_PSW}
        docker build -t ${IMAGE_NAME}:${TAG} .
        docker push ${IMAGE_NAME}:${TAG}
        """
      }
      post {
        success {
           echo "Build & Push Successful"
        }
        failure {
           echo "Build & Push Failed"
        }
      }
    }

    stage('Approval') {
      steps {
         input(
           message: "Deploy application with ${TAG} version to production ?",
           ok: 'Yes, deploy it'
         )
      }
    }

    stage('Update Manifest') {
      steps {
        script {
          withCredentials([usernamePassword(credentialsId: 'github-cred',
               usernameVariable: 'username',
               passwordVariable: 'password')]){
                sh("""
                rm -rf simple-webapp-manifest
                git clone $APP_MANIFEST_REPO
                cd simple-webapp-manifest
                sed -i "s/webapp:.*/webapp:$TAG/g" deployment.yaml
                git config --global user.email "example@main.com"
                git config --global user.name "example"
                git add . && git commit -m 'update image tag'
                git push https://$username:$password@github.com$APP_MANIFEST_PATH main
                """)
          }
        }
      }
    }

    stage("Sync App ArgoCD"){
      steps{
        script {
          withCredentials([string(credentialsId: 'argocd-cred', variable: 'ARGOCD_AUTH_TOKEN')]){
                sh("""
                export ARGOCD_SERVER='$ARGOCD_SERVER'
                export ARGOCD_OPTS='$ARGOCD_OPTS'
                argocd app sync simple-webapp
                """)
          }
        }
      }
    }
  }
}
