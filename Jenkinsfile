pipeline {
  agent any

  environment {
    // hostname or ip of the server to reboot
    TARGET = "3.110.47.25"
    TARGET_USER = "ec2-user"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Pre-checks') {
      steps {
        echo "Checking connectivity to ${env.TARGET}"

        // <-- integrated snippet starts here
        steps {
          sshagent(credentials: ['your-ssh-cred-id']) {
            sh "ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${env.TARGET_USER}@${env.TARGET} 'hostname || true'"
          }
        }
        // <-- integrated snippet ends here

        // Note: the original single-line ssh check has been replaced by the sshagent-wrapped command above.
        // If you want to keep both checks, move the original sh call below this block.
      }
    }

    stage('Notify: Rebooting') {
      steps {
        echo "About to reboot ${env.TARGET}. Make sure this is intended."
      }
    }

    stage('Reboot server (via SSH agent)') {
      steps {
        // uses the ssh-agent plugin with credentials id 'server-ssh'
        sshagent (credentials: ['server-ssh']) {
          // graceful reboot (sudo may require NOPASSWD configured for the user)
          sh """
            ssh -o StrictHostKeyChecking=no ${TARGET_USER}@${TARGET} '
              echo \"Running pre-reboot checks...\"
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
