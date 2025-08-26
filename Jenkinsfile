pipeline {
  agent any

  environment {
    REPO_URL          = 'https://github.com/Harish6498-git/Kubernetes-ArgoCD.git'
    GITHUB_TOKEN_CRED = 'github-token'   // Secret Text credential ID (GitHub PAT with "repo" scope)
  }

  parameters {
    string(name: 'App_Name',        description: 'App name (used to auto-find manifest, e.g. datastore-deploy)')
    string(name: 'App_Version',     description: 'Image tag to deploy, e.g. 1.2')
    string(name: 'Branch',          defaultValue: 'main', description: 'Git branch to push to')
    string(name: 'Manifest_Path',   defaultValue: '',     description: 'Optional exact manifest path if you know it')
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
          set -euo pipefail

          # 1) Resolve the manifest to edit
          FILE=""
          if [ -n "${Manifest_Path}" ] && [ -f "${Manifest_Path}" ]; then
            FILE="${Manifest_Path}"
          else
            # Try common locations based on App_Name
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

          echo "Using manifest: $FILE"

          # 2) Update the image line while preserving indentation
          # Replaces:    image: anything
          # With:        image: harish0604/datastore:${App_Version}
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
            set -euo pipefail

            # Configure git identity & mark workspace safe
            git config user.name  "jenkins"
            git config user.email "jenkins@local"
            git config --global --add safe.directory "$PWD" || true

            # Stage only YAML files that changed
            if git status --porcelain | grep -E '\\.ya?ml$' >/dev/null 2>&1; then
              git add $(git status --porcelain | awk '/\\.ya?ml$/ {print $2}')
            fi

            # Commit iff something is staged
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

