pipeline {
    agent any

    parameters {
        string(name: 'USERNAME', defaultValue: 'herky', description: 'Username for target server(s)')
        text(name: 'TARGET_HOSTS', defaultValue: '10.0.0.5\n10.0.0.6', description: 'List of target server IPs (one per line)')
        string(name: 'TARGET_PATH', defaultValue: '/opt/firmware/', description: 'Destination path on target servers')
    }

    environment {
        FIRMWARE_DIR_REPO = "firmware"
        FIRMWARE_FILE_LOCAL = "${env.WORKSPACE}/firmware"
    }

    triggers {
        pollSCM('* * * * *') // Poll every minute
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

        stage('Copy Firmware Files to Target Servers') {
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
                sshagent(credentials: ['your-jenkins-ssh-credential-id']) {
                    script {
                        def hosts = params.TARGET_HOSTS.split("\n")
                        for (host in hosts) {
                            host = host.trim()
                            if (host) {
                                if (fileExists("${FIRMWARE_DIR_REPO}/harsha")) {
                                    sh """
                                        echo "Transferring harsha to ${host}"
                                        scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${FIRMWARE_DIR_REPO}/harsha ${params.USERNAME}@${host}:${params.TARGET_PATH}
                                    """
                                }
                                if (fileExists("${FIRMWARE_DIR_REPO}/herk")) {
                                    sh """
                                        echo "Transferring herk to ${host}"
                                        scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${FIRMWARE_DIR_REPO}/herk ${params.USERNAME}@${host}:${params.TARGET_PATH}
                                    """
                                }
                            }
                        }
                    }
                }
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
                                    return false
                                }
                            }
                        }
                        return true
                    }
                }
            }
            steps {
                echo "No firmware changes detected."
            }
        }
    }
}
