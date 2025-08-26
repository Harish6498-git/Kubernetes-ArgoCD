pipeline {
  agent any

  environment {
    REPO_URL          = 'https://github.com/Harish6498-git/Kubernetes-ArgoCD.git'
    GITHUB_TOKEN_CRED = 'github-token'   // Secret Text credential ID (GitHub PAT)
  }

  parameters {
    string(name: 'App_Name',      description: 'App name (used to auto-find manifest, e.g. Datastore)')
    string(name: 'App_Version',   description: 'Image tag to deploy, e.g. 1.2')
    string(name: 'Branch',        defaultValue: 'main', description: 'Git branch to push to')
    string(name: 'Manifest_Path', defaultValue: 'Datastore/datastore-deploy.yaml', description: 'Exact manifest path (or leave default)')
  }

  stages {
    stage('Clone repo') {
      steps {
        checkout scmGit(
          branches: [[name: "*/${params.Branch}"]],
          extensions: [],
          userRemoteConfigs: [[url: REPO_URL]]
        )
        sh 'echo "Repo top-level:" && ls -la'
      }
    }

    stage('Update image in manifest') {
      steps {
        sh '''
          set -eu

          # Safe defaults to avoid "parameter not set" under -u
          MP="${Manifest_Path:-}"
          AN="${App_Name:-}"
          AV="${App_Version:-}"

          FILE=""
          if [ -n "$MP" ] && [ -f "$MP" ]; then
            FILE="$MP"
          else
            # Try common locations using App_Name
            for f in "./${AN}.yaml" "./${AN}.yml" "./${AN}/${AN}.yaml" "./${AN}/${AN}.yml"; do
              if [ -f "$f" ]; then FILE="$f"; break; fi
            done
            # Fallback: shallow search for a file named like the app
            if [ -z "$FILE" ]; then
              FILE="$(find . -maxdepth 3 -type f \\( -iname "${AN}.yaml" -o -iname "${AN}.yml" \\) | head -n 1 || true)"
            fi
          fi

          if [ -z "$FILE" ] || [ ! -f "$FILE" ]; then
            echo "ERROR: Could not find manifest for App_Name='${AN}'."
            echo "YAML files in repo (depth<=3):"
            find . -maxdepth 3 -type f -name "*.y*ml" -print
            echo "Tip: set the exact file via the 'Manifest_Path' parameter."
            exit 1
          fi

          echo "$FILE" > .manifest_path
          echo "Using manifest: $FILE"

          # Replace the image value while keeping indentation
          sed -i -E "s|(^[[:space:]]*image:[[:space:]]*).*$|\\1harish0604/datastore:${AV}|" "$FILE"

          echo "Preview changes:"
          git --no-pager diff -- "$FILE" | sed -n '1,200p' || true
        '''
      }
    }

    stage('Commit & push to GitHub') {
      steps {
        withCredentials([string(credentialsId: "${GITHUB_TOKEN_CRED}", variable: 'GITHUB_TOKEN')]) {
          sh '''
            set -eu

            FILE="$(cat .manifest_path 2>/dev/null || true)"
            AN="${App_Name:-app}"
            AV="${App_Version:-}"

            if [ -z "$FILE" ] || [ ! -f "$FILE" ]; then
              echo "ERROR: .manifest_path missing or file not found; aborting commit."
              exit 1
            fi

            # Git identity & safe directory
            git config user.name  "jenkins"
            git config user.email "jenkins@local"
            git config --global --add safe.directory "$PWD" || true

            # Stage only the file we touched
            git add "$FILE" || true

            # Commit if staged changes exist
            if ! git diff --cached --quiet; then
              git commit -m "chore(${AN}): bump image to harish0604/datastore:${AV}"
            else
              echo "No changes to commit."
              exit 0
            fi

            # Push using PAT via x-access-token (fails visibly on error)
            git remote set-url origin "https://x-access-token:${GITHUB_TOKEN}@github.com/Harish6498-git/Kubernetes-ArgoCD.git"
            git push origin HEAD:${Branch}
            echo "Push completed."
          '''
        }
      }
    }
  }

  post {
    always {
      // cleanup helper file from workspace
      sh 'rm -f .manifest_path || true'
    }
  }
}
