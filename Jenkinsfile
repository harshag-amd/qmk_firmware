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
        MATCHED_FILES = ''
        PRODUCTS = ''
        REPO_DIR_PATH = "${WORKSPACE}/repo"
    }

    triggers {
        pollSCM('* * * * *') // every minute
    }

    stages {
        stage('Detect Files in Latest Commit') {
            steps {
                dir(env.REPO_DIR_PATH) {
                    script {
                        def latestCommit = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                        echo "Checking latest commit: ${latestCommit}"

                        def changedFiles = sh(
                            script: "git show --name-only --pretty='' ${latestCommit}",
                            returnStdout: true
                        ).trim().split("\n").collect { it.trim() }.findAll { it }

                        def extensions = params.ALLOWED_EXTENSIONS.split("\n").collect { it.trim() }
                        def products = params.PRODUCT_LIST.split("\n").collect { it.trim() }

                        def matched = [] as Set
                        def foundProducts = [] as Set

                        for (file in changedFiles) {
                            if (!extensions.any { file.endsWith(it) }) continue
                            def found = products.find { p -> file.contains("/${p}/") }
                            if (found) {
                                matched << file
                                foundProducts << found
                            }
                        }

                        env.MATCHED_FILES = matched.join(",")
                        env.PRODUCTS = foundProducts.join(",")

                        if (matched.isEmpty()) {
                            echo "No matching files in latest commit."
                        } else {
                            echo "Matched files:\n${matched.join('\n')}"
                            echo "Detected products: ${foundProducts.join(', ')}"
                        }
                    }
                }
            }
        }

        stage('Push File to Target Server(s)') {
            when {
                expression { return env.MATCHED_FILES?.trim() }
            }
            steps {
                sshagent(credentials: ['jenkins-id']) {
                    dir(env.REPO_DIR_PATH) {
                        script {
                            def files = env.MATCHED_FILES.split(",").collect { it.trim() }
                            def hosts = params.TARGET_HOSTS.split("\n").collect { it.trim() }

                            for (host in hosts) {
                                for (relPath in files) {
                                    def fullPath = "${env.REPO_DIR_PATH}/${relPath}"
                                    def filename = relPath.tokenize("/").last()
                                    if (fileExists(fullPath)) {
                                        echo "Copying ${relPath} to ${host}:${params.TARGET_PATH}/${filename}"
                                        sh """
                                            scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null '${fullPath}' ${params.USERNAME}@${host}:${params.TARGET_PATH}/${filename}
                                        """
                                    } else {
                                        echo "File not found: ${fullPath}"
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Update System Packages') {
            when {
                expression { return env.MATCHED_FILES?.trim() }
            }
            steps {
                sshagent(credentials: ['jenkins-id']) {
                    script {
                        def updateCmd = [
                            apt: "sudo -S apt update -y",
                            dnf: "sudo -S dnf update -y",
                            yum: "sudo -S yum update -y"
                        ][params.PKG_MANAGER]

                        def hosts = params.TARGET_HOSTS.split("\n").collect { it.trim() }

                        for (host in hosts) {
                            echo "Updating ${host} with ${params.PKG_MANAGER}"
                            sh """
                                ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${params.USERNAME}@${host} "${updateCmd}" <<< 'w'
                            """
                        }
                    }
                }
            }
        }

        stage('No Matching Files') {
            when {
                not {
                    expression { return env.MATCHED_FILES?.trim() }
                }
            }
            steps {
                echo "No relevant changes detected in latest commit."
            }
        }
    }
}
