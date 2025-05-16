pipeline {
    agent any

    parameters {
        string(name: 'USERNAME', defaultValue: 'herky', description: 'Username for target server(s)')
        text(name: 'TARGET_HOSTS', defaultValue: '10.86.22.146', description: 'List of target server IPs (one per line)')
        string(name: 'TARGET_PATH', defaultValue: '/opt/firmware/', description: 'Destination path on target servers')
        text(name: 'ALLOWED_EXTENSIONS', defaultValue: '.h\n.c\n.cpp\n.bin', description: 'Allowed file extensions (one per line)')
        choice(name: 'PKG_MANAGER', choices: ['apt', 'dnf', 'yum'], description: 'Package manager to run system update')
        string(name: 'REPO_DIR', defaultValue: '.', description: 'Subdirectory to monitor for file changes (e.g., ip_fw or . for root)')
        text(name: 'PRODUCT_LIST', defaultValue: 'arden\nnavi\nkracken\nraphael\nremambrant', description: 'List of valid products to check in paths')
    }

    environment {
        MATCHED_FILES = ''
        PRODUCTS = ''
    }

    triggers {
        pollSCM('* * * * *') // Poll every minute
    }

    stages {
        stage('Detect Changes') {
            steps {
                dir(params.REPO_DIR) {
                    script {
                        // Get the latest commit ID
                        def latestCommit = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                        echo "Latest commit ID: ${latestCommit}"

                        // Get the files changed in this commit
                        def changedFiles = sh(
                            script: "git show --name-only --pretty='' ${latestCommit}",
                            returnStdout: true
                        ).trim().split("\n").collect { it.trim() }.findAll { it }

                        // Filter files by extension and product folder
                        def extensions = params.ALLOWED_EXTENSIONS.split("\n").collect { it.trim() }
                        def products = params.PRODUCT_LIST.split("\n").collect { it.trim() }

                        def matched = [] as Set
                        def foundProducts = [] as Set

                        for (file in changedFiles) {
                            if (!extensions.any { file.endsWith(it) }) continue
                            def matchedProduct = products.find { product -> file.contains("/${product}/") }
                            if (matchedProduct) {
                                matched << file
                                foundProducts << matchedProduct
                            }
                        }

                        env.MATCHED_FILES = matched.join(",")
                        env.PRODUCTS = foundProducts.join(",")

                        if (matched.isEmpty()) {
                            echo "No matching changes found in the latest commit."
                        } else {
                            echo "Matched files:\n${matched.join('\n')}"
                            echo "Detected products: ${foundProducts.join(', ')}"
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
                dir(params.REPO_DIR) {
                    sshagent(credentials: ['jenkins-id']) {
                        script {
                            def hosts = params.TARGET_HOSTS.split("\n").collect { it.trim() }.findAll { it }
                            def filesToSend = env.MATCHED_FILES.split(",").collect { it.trim() }

                            for (host in hosts) {
                                for (file in filesToSend) {
                                    if (fileExists(file)) {
                                        def filename = file.tokenize("/").last()
                                        sh """
                                            echo "Copying ${file} to ${host}..."
                                            scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null '${file}' ${params.USERNAME}@${host}:${params.TARGET_PATH}/${filename}
                                        """
                                    } else {
                                        echo "Warning: ${file} does not exist locally. Skipping."
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
                            sh """
                                echo "Running ${params.PKG_MANAGER} update on ${host}..."
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
