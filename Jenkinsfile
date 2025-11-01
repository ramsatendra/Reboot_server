pipeline {
  agent any

  environment {
    TARGET = "3.110.47.25"
    TARGET_USER = "ec2-user"
    // change this to the credential id you have in Jenkins (or leave as 'server-ssh' if that's what you use)
    SSH_CRED_ID = "server-ssh"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    // <-- Prep known_hosts stage inserted here
    stage('Prep known_hosts') {
      steps {
        sh '''
          mkdir -p "$HOME/.ssh"
          # add host key (safe) for the build user
          ssh-keyscan -H ${TARGET} >> "$HOME/.ssh/known_hosts" 2>/dev/null || true
          chmod 600 "$HOME/.ssh/known_hosts" || true
          echo "Known hosts for build:"
          sed -n '1,2p' "$HOME/.ssh/known_hosts" || true
        '''
      }
    }

    stage('Pre-checks') {
      steps {
        echo "Checking connectivity to ${env.TARGET}"

        // use ssh-agent plugin to provide private key to ssh
        sshagent(credentials: ['server-ssh']) {
          // now that known_hosts is populated, normal host-key checking will succeed
          sh "ssh -o BatchMode=yes -o ConnectTimeout=5 ${env.TARGET_USER}@${env.TARGET} 'hostname || true'"
        }

        // optional: a short extra check using the same credential id (uncomment if desired)
        // sshagent(credentials: [env.SSH_CRED_ID]) {
        //   sh "ssh -o BatchMode=yes -o ConnectTimeout=5 ${env.TARGET_USER}@${env.TARGET} 'echo pre-check ok || true'"
        // }
      }
    }

    stage('Notify: Rebooting') {
      steps {
        echo "About to reboot ${env.TARGET}. Make sure this is intended."
      }
    }

    stage('Reboot server (via SSH agent)') {
      steps {
        // uses the ssh-agent plugin with credentials id in SSH_CRED_ID
        sshagent (credentials: [env.SSH_CRED_ID]) {
          sh """
            ssh -o BatchMode=yes -o ConnectTimeout=10 ${env.TARGET_USER}@${env.TARGET} '
              echo "Running pre-reboot checks..."
              uptime
              who -q
              sudo /sbin/shutdown -r +0 "Reboot triggered by Jenkins job ${BUILD_NUMBER}"
            '
          """
        }
      }
    }

    stage('Post') {
      steps {
        echo "Reboot command issued. Wait for it to come back and continue if needed."
      }
    }
  }
}
