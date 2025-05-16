pipeline {
    agent any

    parameters {
        string(name: 'USERNAME', defaultValue: 'herky', description: 'Username for target server(s)')
        text(name: 'TARGET_HOSTS', defaultValue: '10.86.22.146', description: 'List of target server IPs (one per line)')
        string(name: 'TARGET_PATH', defaultValue: '/opt/firmware/', description: 'Destination path on target servers')
        text(name: 'ALLOWED_EXTENSIONS', defaultValue: '.h\n.c\n.cpp\n.bin', description: 'Allowed file extensions (one per line)')
        choice(name: 'PKG_MANAGER', choices: ['apt', 'dnf', 'yum'], description: 'Package manager to run system update')
        string(name: 'GIT_REPO_URL', defaultValue: '', description: 'Git repository URL (SSH format)')
        string(name: 'GIT_BRANCH', defaultValue: 'amd/main', description: 'Git branch to track')
        string(name: 'REPO_DIR', defaultValue: 'repo', description: 'Local directory name for repo clone')
        text(name: 'PRODUCT_LIST', defaultValue: 'arden\nnavi\nkracken\nraphael\nremambrant', description: 'List of valid products to check in paths')
    }

    environment {
        MATCHED_FILES = ''
        PRODUCTS = ''
        REPO_DIR_PATH = "${WORKSPACE}/${params.REPO_DIR}"
    }

    triggers {
        pollSCM('H/5 * * * *') // Poll every 5 minutes (adjust as needed)
    }

    stages {
        stage('Detect Changes') {
            steps {
                dir(env.REPO_DIR_PATH) {
                    script {
                        // Get the latest commit hash
                        def latestCommit = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                        echo "Latest Commit: ${latestCommit}"

                        // Get changed files in the latest commit only
                        def changedFilesRaw = sh(
                            script: "git diff-tree --no-commit-id --name-only -r ${latestCommit}",
                            returnStdout: true
                        ).trim()

                        def changedFiles = changedFilesRaw ? changedFilesRaw.split("\n").findAll { it } : []

                        echo "Changed files:\n${changedFiles.join('\n')}"

                        def extensions = params.ALLOWED_EXTENSIONS.split("\n").collect { it.trim() }
                        def products = params.PRODUCT_LIST.split("\n").collect { it.trim() }

                        def matched = [] as Set
                        def foundProducts = [] as Set

                        for (file in changedFiles) {
                            if (!extensions.any { file.endsWith(it) }) {
                                continue
                            }
                            def matchedProduct = products.find { product -> file.contains("/${product}/") }
                            if (matchedProduct) {
                                matched << file
                                foundProducts << matchedProduct
                            }
                        }

                        if (matched.isEmpty()) {
                            echo "No matching changes found."
                        } else {
                            echo "Matched Files:\n${matched.join('\n')}"
                            echo "Detected Products: ${foundProducts.join(', ')}"
                        }

                        // Save matched files and products to env variables for use in later stages
                        env.MATCHED_FILES = matched.join(',')
                        env.PRODUCTS = foundProducts.join(',')
                    }
                }
            }
        }

        stage('Copy Changed Files to Target') {
            when {
                expression { return env.MATCHED_FILES?.trim() }
            }
            steps {
                dir(env.REPO_DIR_PATH) {
                    sshagent(credentials: ['jenkins-id']) {
                        script {
                            def hosts = params.TARGET_HOSTS.split("\n").collect { it.trim() }.findAll { it }
                            def filesToCopy = env.MATCHED_FILES.split(",").collect { it.trim() }

                            for (host in hosts) {
                                for (file in filesToCopy) {
                                    if (fileExists(file)) {
                                        def filename = file.tokenize("/").last()
                                        sh """
                                            echo "Copying file ${file} to ${host}:${params.TARGET_PATH} ..."
                                            scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${file} ${params.USERNAME}@${host}:${params.TARGET_PATH}/${filename}
                                        """
                                    } else {
                                        echo "Warning: file ${file} not found on workspace."
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
                                updateCmd = "sudo apt update -y"
                                break
                            case "dnf":
                                updateCmd = "sudo dnf update -y"
                                break
                            case "yum":
                                updateCmd = "sudo yum update -y"
                                break
                            default:
                                error("Unsupported package manager: ${params.PKG_MANAGER}")
                        }

                        for (host in hosts) {
                            sh """
                                echo "Running ${params.PKG_MANAGER} update on ${host}..."
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
                    expression { return env.MATCHED_FILES?.trim() }
                }
            }
            steps {
                echo "No relevant changes detected. Skipping file copy and server update."
            }
        }
    }
}
