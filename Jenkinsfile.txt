pipeline {
  agent any

  environment {
    // hostname or ip of the server to reboot (keep this in credentials/config as appropriate)
    TARGET = "10.0.1.23"
    TARGET_USER = "ubuntu"
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
        sh "ssh -o BatchMode=yes -o ConnectTimeout=5 ${env.TARGET_USER}@${env.TARGET} 'hostname || true'"
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
