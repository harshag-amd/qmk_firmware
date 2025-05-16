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
        text(name: 'PRODUCT_LIST', defaultValue: 'arden\nnavi\nkracken\nraphael\nremambrant', description: 'List of valid products to check in paths')
    }

    environment {
        REPO_DIR_PATH = "${WORKSPACE}/${params.REPO_DIR}"
    }

    // Groovy variables to hold matched files and products
    // These are accessible across stages for condition checks
    def matchedFilesGlobal = []
    def detectedProductsGlobal = []

    triggers {
        pollSCM('* * * * *')
    }

    stages {
        stage('Detect Changes') {
            steps {
                dir(env.REPO_DIR_PATH) {
                    script {
                        def latestCommit = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                        echo "Latest Commit: ${latestCommit}"

                        def changedFilesStr = sh(
                            script: "git diff-tree --no-commit-id --name-only -r ${latestCommit}",
                            returnStdout: true
                        ).trim()

                        echo "Changed files:\n${changedFilesStr}"

                        def changedFiles = changedFilesStr.split("\n").findAll { it }
                        def extensions = params.ALLOWED_EXTENSIONS.split("\n").collect { it.trim() }
                        def products = params.PRODUCT_LIST.split("\n").collect { it.trim().toLowerCase() }

                        def matchedFiles = [] as Set
                        def detectedProducts = [] as Set

                        for (file in changedFiles) {
                            if (!extensions.any { ext -> file.endsWith(ext) }) {
                                continue
                            }

                            def matchedProduct = products.find { product -> file.toLowerCase().contains(product) }
                            if (matchedProduct) {
                                matchedFiles << file
                                detectedProducts << matchedProduct
                            }
                        }

                        matchedFilesGlobal = matchedFiles as List
                        detectedProductsGlobal = detectedProducts as List

                        if (matchedFilesGlobal.isEmpty()) {
                            echo "No matching changes found."
                        } else {
                            echo "Matched files:\n${matchedFilesGlobal.join('\n')}"
                            echo "Detected products: ${detectedProductsGlobal.join(', ')}"
                        }
                    }
                }
            }
        }

        stage('Copy Changed Files to Target') {
            when {
                expression { return matchedFilesGlobal && matchedFilesGlobal.size() > 0 }
            }
            steps {
                dir(env.REPO_DIR_PATH) {
                    sshagent(credentials: ['your-jenkins-ssh-credential-id']) {
                        script {
                            def hosts = params.TARGET_HOSTS.split("\n").collect { it.trim() }.findAll { it }

                            for (host in hosts) {
                                for (file in matchedFilesGlobal) {
                                    if (fileExists(file)) {
                                        def filename = file.tokenize("/").last()
                                        echo "Copying ${file} to ${host}:${params.TARGET_PATH}/${filename}..."
                                        sh """
                                            scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${file} ${params.USERNAME}@${host}:${params.TARGET_PATH}/${filename}
                                        """
                                    } else {
                                        echo "Warning: ${file} not found locally."
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Update Servers') {
            when {
                expression { return matchedFilesGlobal && matchedFilesGlobal.size() > 0 }
            }
            steps {
                sshagent(credentials: ['your-jenkins-ssh-credential-id']) {
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
                            echo "Running ${params.PKG_MANAGER} update on ${host}..."
                            sh """
                                ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${params.USERNAME}@${host} "${updateCmd}"
                            """
                        }
                    }
                }
            }
        }

        stage('No Relevant Changes') {
            when {
                not {
                    expression { return matchedFilesGlobal && matchedFilesGlobal.size() > 0 }
                }
            }
            steps {
                echo "No relevant changes detected. Skipping file copy and server update."
            }
        }
    }
}
