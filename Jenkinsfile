pipeline {
    agent any

    environment {
        FIRMWARE_DIR_REPO = "firmware"
        FIRMWARE_FILE_LOCAL = "/opt/firmware"
    }

    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/harshag-amd/qmk_firmware.git', branch: 'master'
            }
        }

        stage('Copy Firmware') {
            when {
                changeset {
                    file '**/firmware/harsha'
                    file '**/firmware/herk'
                }
            }
            steps {
                sh '''
                    mkdir -p $FIRMWARE_FILE_LOCAL
                    cp $FIRMWARE_DIR_REPO/harsha $FIRMWARE_FILE_LOCAL/harsha 2>/dev/null || true
                    cp $FIRMWARE_DIR_REPO/herk $FIRMWARE_FILE_LOCAL/herk 2>/dev/null || true
                    echo "Firmware files copied to $FIRMWARE_FILE_LOCAL"
                '''
            }
        }

        stage('Flash Firmware') {
            when {
                changeset {
                    file '**/firmware/harsha'
                    file '**/firmware/herk'
                }
            }
            steps {
                sh '''
                    chmod +x /usr/local/bin/flash_apu.sh
                    /usr/local/bin/flash_apu.sh $FIRMWARE_FILE_LOCAL/harsha || true
                    /usr/local/bin/flash_apu.sh $FIRMWARE_FILE_LOCAL/herk || true
                '''
            }
        }

        stage('No Firmware Change') {
            when {
                not {
                    changeset {
                        file '**/firmware/harsha'
                        file '**/firmware/herk'
                    }
                }
            }
            steps {
                echo "No firmware update (harsha/herk) detected — skipping flash."
            }
        }
    }
}
