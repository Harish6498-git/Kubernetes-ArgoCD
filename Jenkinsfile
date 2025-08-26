pipeline {
  agent any 
  environment {
    GITHUB_TOKEN=credentials("github-token")
  }
  parameters {
    string(name: "App_Name", description: "application name that need to deployed")
    string(name: "App_Version", description: "version of the application that need to be deployed")
  }
  stages {
    stage("cloning repo") {
      steps {
        checkout scmGit(branches: [[name : '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/Harish6498-git/Kubernetes-ArgoCD.git']])
      }
    }
    stage("Updating image version in a file") {
      steps {
        sh """
          echo "---------------- Updating File Context -----------------"
          cd "${params.App_Name}"
          sed -i 's/image:.*/image: harish0604\\/datastore:${params.App_Version}/g' "{params.App_Name}".yaml
          echo "--------------- File Content Updated Successfully -------------"
        """
      }
    }
    stage("pushing code to github") {
      steps {
        sh """
          echo "------------------ Pushing Changes To Github --------------------"
          git add .
          git commit -m "docker image updated"
          git push https://$GITHUB_TOKEN@github.com/Harish6498-git/Kubernetes-ArgoCD.git HEAD:main
          echo "------------------ Pushed Changes Successfully --------------------"
        """
      }
    }
  }
}
