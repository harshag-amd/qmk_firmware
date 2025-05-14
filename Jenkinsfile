pipeline {
    agent any

    environment {
        FIRMWARE_REPO_DIR = "firmware"
        FIRMWARE_LOCAL_DIR = "${WORKSPACE}/firmware"
        DEST_USER = "herky"
        DEST_HOST = "10.86.22.146"
        DEST_PATH = "/Downloads/firmware"
        SCP_OPTIONS = "-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
    }

    triggers {
        pollSCM('* * * * *')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Prepare & Transfer Firmware') {
            steps {
                script {
                    def hasHarsha = fileExists("${FIRMWARE_REPO_DIR}/harsha")
                    def hasHerk = fileExists("${FIRMWARE_REPO_DIR}/herk")

                    if (!hasHarsha && !hasHerk) {
                        echo "No firmware changes detected — skipping transfer."
                        currentBuild.result = 'SUCCESS'
                        return
                    }
                }

                sh '''
                    mkdir -p $FIRMWARE_LOCAL_DIR

                    if [ -f "$FIRMWARE_REPO_DIR/harsha" ]; then
                        echo "Copying harsha..."
                        cp $FIRMWARE_REPO_DIR/harsha $FIRMWARE_LOCAL_DIR/harsha
                    fi

                    if [ -f "$FIRMWARE_REPO_DIR/herk" ]; then
                        echo "Copying herk..."
                        cp $FIRMWARE_REPO_DIR/herk $FIRMWARE_LOCAL_DIR/herk
                    fi
                '''

                sshagent(credentials: ['your-jenkins-ssh-credential-id']) {
                    sh '''
                        if [ -f "$FIRMWARE_LOCAL_DIR/harsha" ]; then
                            echo "SCP harsha..."
                            scp $SCP_OPTIONS $FIRMWARE_LOCAL_DIR/harsha $DEST_USER@$DEST_HOST:$DEST_PATH
                        fi

                        if [ -f "$FIRMWARE_LOCAL_DIR/herk" ]; then
                            echo "SCP herk..."
                            scp $SCP_OPTIONS $FIRMWARE_LOCAL_DIR/herk $DEST_USER@$DEST_HOST:$DEST_PATH
                        fi
                    '''
                }
            }
        }

        stage('Flash Firmware on Remote APU') {
            steps {
                sshagent(credentials: ['your-jenkins-ssh-credential-id']) {
                    sh '''
                        echo "Running remote firmware flash..."
                        ssh $SCP_OPTIONS $DEST_USER@$DEST_HOST 'sudo /usr/local/bin/flash_apu.sh'
                    '''
                }
            }
        }
    }
}
