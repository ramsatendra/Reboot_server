pipeline {
  agent any

  environment {
    TARGET = "3.110.47.25"
    TARGET_USER = "ec2-user"
    // Set this to your Jenkins credential id (SSH Username with private key)
    CRED_ID = "server-ssh"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Prep known_hosts') {
      steps {
        // create per-build known_hosts so SSH won't prompt
        sh '''
          mkdir -p "$HOME/.ssh"
          ssh-keyscan -H ${TARGET} >> "$HOME/.ssh/known_hosts" 2>/dev/null || true
          chmod 600 "$HOME/.ssh/known_hosts" || true
          echo "Known hosts entries (first 5 lines):"
          sed -n '1,5p' "$HOME/.ssh/known_hosts" || true
        '''
      }
    }

    stage('Pre-checks') {
      steps {
        echo "Checking connectivity to ${env.TARGET}"

        // withCredentials writes the private key to a temp file accessible via $KEYFILE
        withCredentials([sshUserPrivateKey(credentialsId: "${CRED_ID}",
                                           keyFileVariable: 'KEYFILE',
                                           usernameVariable: 'SSHUSER')]) {
          sh '''
            set -o pipefail
            chmod 600 "$KEYFILE" || true

            # show the temp key path for debugging (optional)
            echo "Using keyfile: $KEYFILE"
            ls -l "$KEYFILE" || true

            # ensure known_hosts populated (Prep stage should have done it)
            mkdir -p "$HOME/.ssh"
            ssh-keyscan -H ${TARGET} >> "$HOME/.ssh/known_hosts" 2>/dev/null || true

            # run an SSH connectivity test using the temp key file
            ssh -o BatchMode=yes -o ConnectTimeout=8 -i "$KEYFILE" ${TARGET_USER}@${TARGET} 'hostname || true'
          '''
        }
      }
    }

    stage('Notify: Rebooting') {
      steps {
        echo "About to reboot ${env.TARGET}. Make sure this is intended."
      }
    }

    stage('Reboot server (via credentials)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: "${CRED_ID}",
                                           keyFileVariable: 'KEYFILE',
                                           usernameVariable: 'SSHUSER')]) {
          sh '''
            chmod 600 "$KEYFILE" || true

            # Reboot - using BatchMode so the ssh command fails non-interactively if auth fails
            ssh -o BatchMode=yes -o ConnectTimeout=10 -i "$KEYFILE" ${TARGET_USER}@${TARGET} <<'EOSSH'
              echo "Running pre-reboot checks..."
              uptime
              who -q
              # sudo may need NOPASSWD; uncomment the actual reboot once you are ready
              sudo /sbin/shutdown -r +0 "Reboot triggered by Jenkins job ${BUILD_NUMBER}"
EOSSH
          '''
        }
      }
    }

    stage('Post') {
      steps {
        echo "Reboot command issued. Wait for it to come back and continue if needed."
      }
    }
  }

  post {
    failure {
      echo "Pipeline failed — check console output for SSH diagnostics."
    }
    success {
      echo "Pipeline completed successfully (reboot command issued)."
    }
  }
}
