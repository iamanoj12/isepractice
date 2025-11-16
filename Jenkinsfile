pipeline {
  agent any

  environment {
    GIT_CREDENTIALS = 'github-pat'                // update if you used a different credentials ID
    REPO_URL = 'https://github.com/iamanoj12/isepractice.git' // adjust
    // Default TARGET_DIR for Windows example; adjust as needed
    TARGET_DIR_WIN = 'C:\Users\DELL\Desktop\portfolio'         // Windows target (adjust)
    TARGET_DIR_UNIX = "${env.WORKSPACE}/deployed-site" // safe default for Unix
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM',
          branches: [[name: '*/master']],
          userRemoteConfigs: [[
            url: env.REPO_URL,
            credentialsId: env.GIT_CREDENTIALS
          ]]
        ])
      }
    }

    stage('Build (none)') {
      steps {
        echo "Static site — no build step."
      }
    }

    stage('Deploy') {
      steps {
        script {
          if (isUnix()) {
            echo "Detected Unix agent. Deploying to ${env.TARGET_DIR_UNIX}"
            sh """
              set -e
              rm -rf "${TARGET_DIR_UNIX}"
              mkdir -p "${TARGET_DIR_UNIX}"
              cp -r ./* "${TARGET_DIR_UNIX}/"
              echo "Deployed to ${TARGET_DIR_UNIX}"
            """
          } else {
            // Windows branch uses bat commands and proper escaping
            def td = TARGET_DIR_WIN
            echo "Detected Windows agent. Deploying to ${td}"
            bat """
              if exist "${td}" rmdir /s /q "${td}"
              mkdir "${td}"
              xcopy /E /I /Y "%WORKSPACE%\\*" "${td}\\\\"
              echo Deployed to ${td}
            """
          }
        }
      }
    }

    stage('Post deploy (status)') {
      steps {
        echo "Deployment finished. Check the TARGET_DIR for files."
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '**/*.html, **/*.css', fingerprint: true
      echo "Archived HTML/CSS artifacts."
    }
  }
}
