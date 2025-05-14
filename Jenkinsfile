pipeline {
    agent any

    environment {
        FIRMWARE_DIR_REPO = "firmware"
        FIRMWARE_FILE_LOCAL = "/opt/firmware"
        DEST_USER = "user"
        DEST_HOST = "10.0.0.5"
        DEST_PATH = "/opt/firmware/"
    }

    triggers {
        pollSCM('* * * * *') // every 5 minutes
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/harshag-amd/qmk_firmware.git', branch: 'master'
            }
        }

        stage('Detect Changes') {
            when {
                expression {
                    def changeLog = currentBuild.changeSets
                    def changed = false
                    for (change in changeLog) {
                        for (file in change.items.collectMany { it.affectedFiles }) {
                            if (file.path == "firmware/harsha" || file.path == "firmware/herk") {
                                changed = true
                            }
                        }
                    }
                    return changed
                }
            }
            steps {
                echo "Firmware change detected"
            }
        }

        stage('Download and Copy Firmware') {
            when {
                expression {
                    def changeLog = currentBuild.changeSets
                    def changed = false
                    for (change in changeLog) {
                        for (file in change.items.collectMany { it.affectedFiles }) {
                            if (file.path == "firmware/harsha" || file.path == "firmware/herk") {
                                changed = true
                            }
                        }
                    }
                    return changed
                }
            }
            steps {
                sh '''
                    sudo mkdir -p $FIRMWARE_FILE_LOCAL

                    if [ -f "$FIRMWARE_DIR_REPO/harsha" ]; then
                        echo "Copying harsha..."
                        cp $FIRMWARE_DIR_REPO/harsha $FIRMWARE_FILE_LOCAL/harsha
                        scp $FIRMWARE_FILE_LOCAL/harsha $DEST_USER@$DEST_HOST:$DEST_PATH
                    fi

                    if [ -f "$FIRMWARE_DIR_REPO/herk" ]; then
                        echo "Copying herk..."
                        cp $FIRMWARE_DIR_REPO/herk $FIRMWARE_FILE_LOCAL/herk
                        scp $FIRMWARE_FILE_LOCAL/herk $DEST_USER@$DEST_HOST:$DEST_PATH
                    fi
                '''
            }
        }

        stage('No Change Detected') {
            when {
                not {
                    expression {
                        def changeLog = currentBuild.changeSets
                        for (change in changeLog) {
                            for (file in change.items.collectMany { it.affectedFiles }) {
                                if (file.path == "firmware/harsha" || file.path == "firmware/herk") {
                                    return false // change found
                                }
                            }
                        }
                        return true // no change found
                    }
                }
            }
            steps {
                echo "No firmware change in harsha or herk."
            }
        }
    }
}
