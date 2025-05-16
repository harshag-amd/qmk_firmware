pipeline {
    agent any

    parameters {
        string(name: 'USERNAME', defaultValue: 'herky', description: 'Username for target server(s)')
        text(name: 'TARGET_HOSTS', defaultValue: '10.86.22.146', description: 'List of target server IPs (one per line)')
        string(name: 'TARGET_PATH', defaultValue: '/opt/firmware/', description: 'Destination path on target servers')
        text(name: 'ALLOWED_EXTENSIONS', defaultValue: '.h\n.c\n.cpp\n.bin', description: 'Allowed file extensions (one per line)')
        choice(name: 'PKG_MANAGER', choices: ['apt', 'dnf', 'yum'], description: 'Package manager to run system update')
        string(name: 'GIT_REPO_URL', defaultValue: 'git@github.amd.com:AMD-Radeon-Driver/drivers.git', description: 'Git repository URL (SSH format)')
        string(name: 'GIT_BRANCH', defaultValue: 'amd/main', description: 'Git branch to track')
        string(name: 'REPO_DIR', defaultValue: '.', description: 'Subdirectory to monitor for file changes (e.g., ip_fw or . for root)')
        text(name: 'PRODUCT_LIST', defaultValue: 'arden\nnavi\nkraken\nraphael\nrembrandt', description: 'List of valid products to check in paths')
    }

    environment {
        MATCHED_FILES = ''
        PRODUCTS = ''
        REPO_DIR_PATH = "${WORKSPACE}/repo"
    }

    triggers {
        pollSCM('* * * * *') // Poll every minute
    }

    stages {
        stage('Detect Changes') {
            steps {
                dir(env.REPO_DIR_PATH) {
                    script {
                        def previousCommit = sh(script: 'git rev-parse HEAD~1', returnStdout: true).trim()
                        def currentCommit = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()

                        echo "Comparing changes from ${previousCommit} to ${currentCommit}"
                        def diffOutput = sh(script: "git diff --name-only ${previousCommit} ${currentCommit}", returnStdout: true).trim()

                        def changedFiles = diffOutput.split("\n").findAll { it } as Set
                        def extensions = params.ALLOWED_EXTENSIONS.split("\n").collect { it.trim() }
                        def productList = params.PRODUCT_LIST.split("\n").collect { it.trim() }

                        def matchedFiles = [] as Set
                        def detectedProducts = [] as Set

                        for (file in changedFiles) {
                            if (!extensions.any { file.endsWith(it) }) continue

                            def foundProduct = productList.find { product -> file.contains("/${product}/") }
                            if (foundProduct) {
                                matchedFiles << file
                                detectedProducts << foundProduct
                            }
                        }

                        env.MATCHED_FILES = matchedFiles.join(",")
                        env.PRODUCTS = detectedProducts.join(",")

                        if (matchedFiles.isEmpty()) {
                            echo "No matching changes found."
                        } else {
                            echo "Matched files:\n${matchedFiles.join('\n')}"
                            echo "Detected products: ${detectedProducts.join(', ')}"
                        }
                    }
                }
            }
        }

        stage('Copy Changed Files') {
            when {
                expression { return env.MATCHED_FILES?.trim() }
            }
            steps {
                sshagent(credentials: ['jenkins-id']) {
                    script {
                        def hosts = params.TARGET_HOSTS.split("\n").collect { it.trim() }
                        def filesToSend = env.MATCHED_FILES.split(",").collect { it.trim() }

                        for (host in hosts) {
                            for (relativePath in filesToSend) {
                                def fullPath = "${env.REPO_DIR_PATH}/${relativePath}"
                                if (fileExists(fullPath)) {
                                    def filename = relativePath.tokenize("/").last()
                                    echo "Copying ${relativePath} to ${host}:${params.TARGET_PATH}/${filename}"
                                    sh """
                                        scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null '${fullPath}' ${params.USERNAME}@${host}:${params.TARGET_PATH}/${filename}
                                    """
                                } else {
                                    echo "Warning: File not found: ${fullPath}"
                                    sh "ls -l '${fullPath}' || echo 'File does not exist'"
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Update Servers') {
            when {
                expression { return env.MATCHED_FILES?.trim() }
            }
            steps {
                sshagent(credentials: ['jenkins-id']) {
                    script {
                        def hosts = params.TARGET_HOSTS.split("\n").collect { it.trim() }
                        def updateCmd = ""

                        switch (params.PKG_MANAGER) {
                            case "apt":
                                updateCmd = "sudo -S apt update -y"
                                break
                            case "dnf":
                                updateCmd = "sudo -S dnf update -y"
                                break
                            case "yum":
                                updateCmd = "sudo -S yum update -y"
                                break
                        }

                        for (host in hosts) {
                            echo "Running ${params.PKG_MANAGER} update on ${host}"
                            sh """
                                ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${params.USERNAME}@${host} "${updateCmd}" <<< 'w'
                            """
                        }
                    }
                }
            }
        }

        stage('No Change Detected') {
            when {
                not {
                    expression { return env.MATCHED_FILES?.trim() }
                }
            }
            steps {
                echo "No relevant changes detected. Skipping file transfer and update."
            }
        }
    }
}
