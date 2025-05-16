pipeline {
    agent any

    parameters {
        string(name: 'USERNAME', defaultValue: 'herky', description: 'Username for target server(s)')
        text(name: 'TARGET_HOSTS', defaultValue: '10.86.22.146', description: 'List of target server IPs (one per line)')
        string(name: 'TARGET_PATH', defaultValue: '/opt/firmware/', description: 'Destination path on target servers')
        text(name: 'ALLOWED_EXTENSIONS', defaultValue: '.h\n.c\n.cpp\n.bin', description: 'Allowed file extensions (one per line)')
        choice(name: 'PKG_MANAGER', choices: ['apt', 'dnf', 'yum'], description: 'Package manager to run system update')
        string(name: 'REPO_DIR', defaultValue: '.', description: 'Subdirectory where repo is checked out (e.g., ip_fw or .)')
        text(name: 'PRODUCT_LIST', defaultValue: 'arden\nnavi\nkracken\nraphael\nremambrant', description: 'Valid products to match in paths')
    }

    environment {
        MATCHED_FILES = ''
        PRODUCTS = ''
    }

    triggers {
        pollSCM('* * * * *') // every minute
    }

    stages {
        stage('Detect Changes') {
            steps {
                dir(params.REPO_DIR) {
                    script {
                        def latestCommit = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                        echo "Latest Commit: ${latestCommit}"

                        def changedFiles = sh(
                            script: "git diff-tree --no-commit-id --name-only -r ${latestCommit}",
                            returnStdout: true
                        ).trim().split("\n").findAll { it }

                        echo "Changed files:\n${changedFiles.join('\n')}"

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

                        env.MATCHED_FILES = matched.join(',')
                        env.PRODUCTS = foundProducts.join(',')

                        if (matched) {
                            echo "Matched Files:\n${matched.join('\n')}"
                            echo "Detected Products: ${foundProducts.join(', ')}"
                        } else {
                            echo "No matching changes."
                        }
                    }
                }
            }
        }

        stage('Copy Changed Files to Target') {
            when {
                expression { return env.MATCHED_FILES?.trim() }
            }
            steps {
                dir(params.REPO_DIR) {
                    sshagent(credentials: ['your-jenkins-ssh-credential-id']) {
                        script {
                            def hosts = params.TARGET_HOSTS.split("\n").collect { it.trim() }.findAll { it }
                            def files = env.MATCHED_FILES.split(',').collect { it.trim() }

                            for (host in hosts) {
                                for (file in files) {
                                    // Make sure directory structure exists
                                    def fullPath = "${params.REPO_DIR}/${file}".replaceFirst('^\\./', '')

                                    if (fileExists(file)) {
                                        echo "Copying file ${file} to ${host}"
                                        def filename = file.tokenize('/').last()
                                        sh """
                                            scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null '${file}' ${params.USERNAME}@${host}:${params.TARGET_PATH}/${filename}
                                        """
                                    } else {
                                        echo "ERROR: File ${file} does not exist in ${params.REPO_DIR}"
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

        stage('No Relevant Changes') {
            when {
                not {
                    expression { return env.MATCHED_FILES?.trim() }
                }
            }
            steps {
                echo "No matching files changed. Skipping file transfer and server update."
            }
        }
    }
}
