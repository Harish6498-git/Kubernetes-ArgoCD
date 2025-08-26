pipeline {
  agent any

  environment {
    REPO_URL          = 'https://github.com/Harish6498-git/Kubernetes-ArgoCD.git'
    GITHUB_TOKEN_CRED = 'github-token'   // Secret Text credential ID (GitHub PAT with "repo" scope)
  }

  parameters {
    string(name: 'App_Name',      description: 'App name (used to auto-find manifest, e.g. Datastore)')
    string(name: 'App_Version',   description: 'Image tag to deploy, e.g. 1.2')
    string(name: 'Branch',        defaultValue: 'main', description: 'Git branch to push to')
    string(name: 'Manifest_Path', defaultValue: '',     description: 'Optional exact manifest path if known')
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

          # Decide which manifest to edit
          FILE=""
          if [ -n "${Manifest_Path}" ] && [ -f "${Manifest_Path}" ]; then
            FILE="${Manifest_Path}"
          else
            # Try common locations using App_Name
            for f in "./${App_Name}.yaml" "./${App_Name}.yml" "./${App_Name}/${App_Name}.yaml" "./${App_Name}/${App_Name}.yml"; do
              if [ -f "$f" ]; then FILE="$f"; break; fi
            done
            # Fallback: shallow search for a file named like the app
            if [ -z "$FILE" ]; then
              FILE="$(find . -maxdepth 3 -type f \\( -iname "${App_Name}.yaml" -o -iname "${App_Name}.yml" \\) | head -n 1 || true)"
            fi
          fi

          if [ -z "$FILE" ] || [ ! -f "$FILE" ]; then
            echo "ERROR: Could not find manifest for App_Name='${App_Name}'."
            echo "YAML files in repo (depth<=3):"
            find . -maxdepth 3 -type f -name "*.y*ml" -print
            echo "Tip: set the exact file via the 'Manifest_Path' parameter."
            exit 1
          fi

          echo "$FILE" > .manifest_path
          echo "Using manifest: $FILE"

          # Replace the image value while keeping indentation
          # image: <anything>  ->  image: harish0604/datastore:${App_Version}
          sed -i -E "s|(^[[:space:]]*image:[[:space:]]*).*$|\\1harish0604/datastore:${App_Version}|" "$FILE"

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
              git commit -m "chore(${App_Name}): bump image to harish0604/datastore:${App_Version}"
            else
              echo "No changes to commit."
            fi

            # Push using PAT via x-access-token
            git remote set-url origin "https://x-access-token:${GITHUB_TOKEN}@github.com/Harish6498-git/Kubernetes-ArgoCD.git"
            git push origin HEAD:${Branch} || true

            echo "Push completed."
          '''
        }
      }
    }
  }
}

