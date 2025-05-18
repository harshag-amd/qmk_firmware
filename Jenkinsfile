pipeline {
    agent any

    /***************
     * PARAMETERS
     ***************/
    parameters {
        string(name: 'DEPLOY_USER', defaultValue: 'herky', description: 'Username for target servers')
        text(name: 'TARGET_SERVER_IPS', defaultValue: '10.86.22.146', description: 'Target server IPs (one per line)')
        string(name: 'DEPLOY_DIRECTORY', defaultValue: '/opt/firmware/', description: 'Target directory on remote servers')
        text(name: 'ALLOWED_FILE_EXTENSIONS', defaultValue: '.h\n.c\n.bin', description: 'Allowed file extensions (one per line)')
        choice(name: 'SYSTEM_PACKAGE_MANAGER', choices: ['apt', 'dnf', 'yum'], description: 'Package manager for system updates')
        string(name: 'SOURCE_REPOSITORY_URL', defaultValue: 'https://github.com/harshag-amd/qmk_firmware.git', description: 'Git repository URL')
        string(name: 'SOURCE_BRANCH', defaultValue: 'master', description: 'Branch to monitor')
        string(name: 'MONITORED_SUBDIRECTORY', defaultValue: '.', description: 'Subdirectory to watch for changes')
    }

    /***************
     * ENVIRONMENT
     ***************/
    environment {
        TRACKING_COMMIT_FILE = "${WORKSPACE}/repo/.last_successful_commit"
        CLONE_DIRECTORY = "${WORKSPACE}/repo"
    }

    /***************
     * TRIGGERS
     ***************/
    triggers {
        pollSCM('* * * * *') // Poll Git repo every minute
    }

    /***************
     * PIPELINE STAGES
     ***************/
    stages {

        stage('🧪 Validate Input Parameters') {
            steps {
                script {
                    if (!params.ALLOWED_FILE_EXTENSIONS?.trim()) {
                        error "ALLOWED_FILE_EXTENSIONS must not be empty!"
                    }
                    if (!params.TARGET_SERVER_IPS?.trim()) {
                        error "TARGET_SERVER_IPS must not be empty!"
                    }
                }
            }
        }
		stage('🔍 Identify Changed Files') {
            steps {
                script {
                    dir(env.CLONE_DIRECTORY) {
                        sh 'git fetch'

                        def lastKnownCommit = fileExists(env.TRACKING_COMMIT_FILE) ? readFile(env.TRACKING_COMMIT_FILE).trim() : ''
                        def currentCommit = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                        def diffRange = lastKnownCommit ? "${lastKnownCommit}..${currentCommit}" : "HEAD~1..HEAD"
                        echo "Checking for file changes between: ${diffRange}"

                        def validExtensions = params.ALLOWED_FILE_EXTENSIONS
                            .split("\n")
                            .collect { it.trim().replaceFirst("^\\.", "") }
                            .findAll { it }
                        if (validExtensions.isEmpty()) {
                            error "No valid file extensions provided. Aborting."
                        }

                        def regexPattern = validExtensions.join("|")
                        echo "Filtering changes with extensions: ${regexPattern}"

                        def changedFilesRaw = sh(
                            script: "git diff --name-only ${diffRange} | grep -Ei '\\.(${regexPattern})\$' || true",
                            returnStdout: true
                        ).trim()

                        if (changedFilesRaw) {
                            def changedFiles = changedFilesRaw.split("\n").findAll { it }
                            writeFile file: 'changed_files.txt', text: changedFiles.join('\n')
                            echo "Detected ${changedFiles.size()} changed file(s):\n" + changedFiles.join("\n")
                        } else {
                            echo "No matching files changed."
                            writeFile file: 'changed_files.txt', text: ""
                        }

                        writeFile file: env.TRACKING_COMMIT_FILE, text: currentCommit
                    }
                }
            }
        }

        stage('🚀 Deploy Modified Files') {
            when {
                expression {
                    fileExists("${env.CLONE_DIRECTORY}/changed_files.txt") &&
                    readFile("${env.CLONE_DIRECTORY}/changed_files.txt").trim()
                }
            }
            steps {
                dir(env.CLONE_DIRECTORY) {
                    script {
                        def changedFiles = readFile('changed_files.txt').trim().split("\n").findAll { it }
                        def targetIPs = params.TARGET_SERVER_IPS.split("\n").findAll { it }

                        sshagent(credentials: ['your-jenkins-ssh-credential-id']) {
                            for (filePath in changedFiles) {
                                if (!fileExists(filePath)) {
                                    error "Missing file: ${filePath}"
                                }

                                for (ip in targetIPs) {
                                    echo "Transferring ${filePath} to ${params.DEPLOY_USER}@${ip}:${params.DEPLOY_DIRECTORY}"
                                    sh """
                                        scp -o StrictHostKeyChecking=no '${filePath}' '${params.DEPLOY_USER}@${ip}:${params.DEPLOY_DIRECTORY}'
                                    """
                                }
                            }
                        }

                        // Clean up after deployment
                        sh 'rm -f changed_files.txt'
                    }
                }
            }
        }
    }
}
