pipeline {
    agent any

    parameters {
        string(name: 'USERNAME', defaultValue: 'herky', description: 'Username for target server(s)')
        text(name: 'TARGET_HOSTS', defaultValue: '10.86.22.146', description: 'List of target server IPs (one per line)')
        string(name: 'TARGET_PATH', defaultValue: '/opt/firmware/', description: 'Destination path on target servers')
        text(name: 'ALLOWED_EXTENSIONS', defaultValue: '.h\n.c\n.bin', description: 'Allowed file extensions (one per line)')
        choice(name: 'PKG_MANAGER', choices: ['apt', 'dnf', 'yum'], description: 'Package manager to run system update')
        string(name: 'GIT_REPO_URL', defaultValue: 'git@github.com:harshag-amd/qmk_firmware.git', description: 'Git repository URL')
        string(name: 'GIT_BRANCH', defaultValue: 'master', description: 'Git branch to track')
        string(name: 'REPO_DIR', defaultValue: 'qmk_firmware', description: 'Subdirectory name for Git clone')
    }

    environment {
        MATCHED_FILES = ''
        IP_BLOCKS = ''
        PRODUCTS = ''
        REPO_DIR_PATH = "${WORKSPACE}/${params.REPO_DIR}"
        COMMIT_TRACK_FILE = "${WORKSPACE}/.jenkins_commit"
    }

    triggers {
        pollSCM('* * * * *') // every minute
    }

    stages {
        stage('Clone or Update Repository') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-personal-jenkins-id', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
                        def repoDir = env.REPO_DIR_PATH

                        if (fileExists(repoDir)) {
                            echo "Repository exists, fetching updates..."
                            dir(repoDir) {
                                sh """
                                    git -c credential.helper='!f() { echo username=\\$GIT_USERNAME; echo password=\\$GIT_TOKEN; }; f' fetch origin ${params.GIT_BRANCH}
                                    git reset --hard origin/${params.GIT_BRANCH}
                                """
                            }
                        } else {
                            echo "Cloning repository..."
                            sh """
                                git -c credential.helper='!f() { echo username=\\$GIT_USERNAME; echo password=\\$GIT_TOKEN; }; f' clone --single-branch --branch ${params.GIT_BRANCH} ${params.GIT_REPO_URL} ${repoDir}
                            """
                        }
                    }
                }
            }
        }

        stage('Detect Changes') {
            steps {
                dir(env.REPO_DIR_PATH) {
                    script {
                        def extensions = params.ALLOWED_EXTENSIONS.split("\n").collect { it.trim() }.findAll { it }
                        def previousCommit = fileExists(env.COMMIT_TRACK_FILE) ? readFile(env.COMMIT_TRACK_FILE).trim() : ''
                        def currentCommit = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                        def changedFiles = []

                        if (previousCommit) {
                            echo "Comparing commits: ${previousCommit} -> ${currentCommit}"
                            changedFiles = sh(script: "git diff --name-only ${previousCommit} ${currentCommit}", returnStdout: true).trim().split("\n")
                        } else {
                            echo "First run or no previous commit found. Using current commit only."
                            changedFiles = sh(script: "git diff-tree --no-commit-id --name-only -r ${currentCommit}", returnStdout: true).trim().split("\n")
                        }

                        writeFile(file: env.COMMIT_TRACK_FILE, text: currentCommit)

                        def matchedFiles = [] as Set
                        def ipBlocks = [] as Set
                        def products = [] as Set

                        for (file in changedFiles) {
                            for (ext in extensions) {
                                if (file.endsWith(ext)) {
                                    matchedFiles << file

                                    def matcher = file =~ /([^\\/]+)\\/([^\\/]+)\\//
                                    if (matcher.find()) {
                                        ipBlocks << matcher.group(1)
                                        products << matcher.group(2)
                                    }
                                    break
                                }
                            }
                        }

                        env.MATCHED_FILES = matchedFiles.join(",")
                        env.IP_BLOCKS = ipBlocks.join(",")
                        env.PRODUCTS = products.join(",")

                        if (matchedFiles.isEmpty()) {
                            echo "No relevant firmware file changes found."
                        } else {
                            echo "Detected changed firmware files:"
                            matchedFiles.each { echo "- ${it}" }
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
                sshagent(credentials: ['your-jenkins-ssh-credential-id']) {
                    script {
                        def hosts = params.TARGET_HOSTS.split("\n").collect { it.trim() }.findAll { it }
                        def changedFiles = env.MATCHED_FILES.split(",").collect { it.trim() }

                        for (host in hosts) {
                            for (file in changedFiles) {
                                def fullPath = "${env.REPO_DIR_PATH}/${file}"
                                if (fileExists(fullPath)) {
                                    def filename = file.tokenize("/").last()
                                    echo "Copying ${fullPath} to ${host}:${params.TARGET_PATH}/${filename}"
                                    sh """
                                        scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null "${fullPath}" ${params.USERNAME}@${host}:${params.TARGET_PATH}/${filename}
                                    """
                                } else {
                                    echo "WARNING: File not found: ${fullPath}"
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
                sshagent(credentials: ['your-jenkins-ssh-credential-id']) {
                    script {
                        def hosts = params.TARGET_HOSTS.split("\n").collect { it.trim() }.findAll { it }
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
                            echo "Updating ${host} using ${params.PKG_MANAGER}..."
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
                echo "No matching firmware changes detected. Skipping copy and update stages."
            }
        }
    }
}
