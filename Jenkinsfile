pipeline {
    agent any

    environment {
        FIRMWARE_FILE_REPO = "qmk_firmware/herk"
        FIRMWARE_FILE_LOCAL = "/opt/herk"
    }

    triggers {
        githubPush() // 👈 Tells Jenkins this is triggered by GitHub webhook
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Copy Firmware') {
            when {
                changeset "**/firmware/firmware-latest.bin"
            }
            steps {
                sh '''
                    mkdir -p $(dirname $FIRMWARE_FILE_LOCAL)
                    cp $FIRMWARE_FILE_REPO $FIRMWARE_FILE_LOCAL
                    echo "Firmware copied to $FIRMWARE_FILE_LOCAL"
                '''
            }
        }

        stage('Flash Firmware') {
            when {
                changeset "**/firmware/firmware-latest.bin"
            }
            steps {
                sh '''
                    chmod +x /usr/local/bin/flash_apu.sh
                    /usr/local/bin/flash_apu.sh $FIRMWARE_FILE_LOCAL
                '''
            }
        }

        stage('No Firmware Change') {
            when {
                not {
                    changeset "**/firmware/firmware-latest.bin"
                }
            }
            steps {
                echo "No firmware update detected — skipping flash."
            }
        }
    }
}
