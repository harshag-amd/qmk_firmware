pipeline {
    agent any

    environment {
        FIRMWARE_REPO_DIR = "firmware"
        FIRMWARE_LOCAL_DIR = "${WORKSPACE}/firmware"
        DEST_USER = "herky"
        DEST_HOST = "10.86.22.146"
        DEST_PATH = "${WORKSPACE}/firmware" // this is just for clarity; used only locally
        SCP_OPTIONS = "-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
    }

    triggers {
        pollSCM('* * * * *') // Poll every minute
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check Firmware Changes') {
            steps {
                script {
                    def hasHarsha = fileExists("${FIRMWARE_REPO_DIR}/harsha")
                    def hasHerk = fileExists("${FIRMWARE_REPO_DIR}/herk")

                    if (!hasHarsha && !hasHerk) {
                        echo "No firmware changes detected — skipping transfer."
                        return
                    }

                    sshagent(credentials: ['your-jenkins-ssh-credential-id']) {
                        if (hasHarsha) {
                            sh """
                                echo "Transferring harsha..."
                                scp $SCP_OPTIONS "$FIRMWARE_REPO_DIR/harsha" "$DEST_USER@$DEST_HOST:$FIRMWARE_REPO_DIR/harsha"
                            """
                        }

                        if (hasHerk) {
                            sh """
                                echo "Transferring herk..."
                                scp $SCP_OPTIONS "$FIRMWARE_REPO_DIR/herk" "$DEST_USER@$DEST_HOST:$FIRMWARE_REPO_DIR/herk"
                            """
                        }
                    }
                }
            }
        }

        stage('Flash Firmware on Remote APU') {
            steps {
                sshagent(credentials: ['your-jenkins-ssh-credential-id']) {
                    sh """
                        echo "Running remote firmware flash..."
                        ssh $SCP_OPTIONS $DEST_USER@$DEST_HOST 'sudo /usr/local/bin/flash_apu.sh'
                    """
                }
            }
        }
    }
}
