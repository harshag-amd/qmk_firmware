pipeline {
    agent any

    parameters {
        string(name: 'USERNAME', defaultValue: 'herky', description: 'Username for target server(s)')
        text(name: 'TARGET_HOSTS', defaultValue: '10.86.22.146', description: 'List of target server IPs (one per line)')
        string(name: 'TARGET_PATH', defaultValue: '/opt/firmware/', description: 'Destination path on target servers')
        text(name: 'FIRMWARE_FILES', defaultValue: 'harsha\nherk', description: 'List of firmware files to monitor (one per line)')
        choice(name: 'PKG_MANAGER', choices: ['apt', 'dnf', 'yum'], description: 'Package manager to run system update')
    }

    environment {
        FIRMWARE_DIR_REPO = "firmware"
    }

    triggers {
        pollSCM('* * * * *') // Every minute
    }

    stages {
        stage('Declarative: Pipeline Intializing') {
            steps {
                echo 'Pipeline initialized.'
            }
        }

        stage('Checkout') {
            steps {
                git url: 'https://github.com/harshag-amd/qmk_firmware.git', branch: 'master'
            }
        }

        stage('Detect Changes') {
            steps {
                script {
                    def changeLog = currentBuild.changeSets
                    def monitoredFiles = params.FIRMWARE_FILES.split("\n").collect { it.trim() }.findAll { it }
                    def changedFiles = []

                    for (change in changeLog) {
                        for (file in change.items.collectMany { it.affectedFiles }) {
                            def fileName = file.path.replaceFirst("^firmware/", "")
                            if (monitoredFiles.contains(fileName)) {
                                changedFiles << fileName
                            }
                        }
                    }

                    env.CHANGED_FILES = changedFiles.join(",")
                    if (changedFiles.isEmpty()) {
                        echo "No monitored firmware files changed."
                    } else {
                        echo "Detected changes in: ${env.CHANGED_FILES}"
                    }
                }
            }
        }

        stage('Copy Changed Firmware Files') {
            when {
                expression { return env.CHANGED_FILES?.trim() }
            }
            steps {
                sshagent(credentials: ['your-jenkins-ssh-credential-id']) {
                    script {
                        def hosts = params.TARGET_HOSTS.split("\n").collect { it.trim() }.findAll { it }
                        def changedFiles = env.CHANGED_FILES.split(",").collect { it.trim() }

                        for (host in hosts) {
                            for (file in changedFiles) {
                                def filePath = "${FIRMWARE_DIR_REPO}/${file}"
                                if (fileExists(filePath)) {
                                    sh """
                                        echo "Copying ${file} to ${host}..."
                                        scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${filePath} ${params.USERNAME}@${host}:${params.TARGET_PATH}
                                    """
                                } else {
                                    echo "Warning: ${filePath} not found."
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Update Servers') {
            when {
                expression { return env.CHANGED_FILES?.trim() }
            }
            steps {
                sshagent(credentials: ['your-jenkins-ssh-credential-id']) {
                    script {
                        def hosts = params.TARGET_HOSTS.split("\n").collect { it.trim() }.findAll { it }
                        def updateCmd = ""

                        switch (params.PKG_MANAGER) {
                            case "apt":
                                updateCmd = "sudo -S apt update -y && sudo -S apt upgrade -y"
                                break
                            case "dnf":
                                updateCmd = "sudo dnf -S update -y"
                                break
                            case "yum":
                                updateCmd = "sudo yum -S update -y"
                                break
                        }

                        for (host in hosts) {
                            sh """
                                echo "Running ${params.PKG_MANAGER} update on ${host}..."
                                echo 'w' | ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${params.USERNAME}@${host} '${updateCmd}'
                            """
                        }
                    }
                }
            }
        }

        stage('No Change Detected') {
            when {
                not {
                    expression { return env.CHANGED_FILES?.trim() }
                }
            }
            steps {
                echo "No firmware changes detected. Skipping copy and update stages."
            }
        }
    }
}
