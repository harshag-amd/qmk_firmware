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
        text(name: 'PRODUCT_KEYWORDS', defaultValue: '''stix
halo
kraken
phoenix
hawk
fire
raphael
rembrandt
gr
navi
vega
raven1
picasso
raven2
zeppelin
threadripper
pinnacle
strix
''', description: 'Keywords to filter relevant products (one per line)')
        string(name: 'SOURCE_REPOSITORY_URL', defaultValue: 'https://github.com/harshag-amd/qmk_firmware.git', description: 'Git repository URL')
        string(name: 'SOURCE_BRANCH', defaultValue: 'master', description: 'Branch to monitor')
        string(name: 'MONITORED_SUBDIRECTORY', defaultValue: '.', description: 'Subdirectory to watch for changes')
        string(name: 'EMAIL_RECIPIENT', defaultValue: 'harshavardhanreddy.gangavaram@amd.com', description: 'Email to send changed files list')
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

         stage('📥 Clone or Update Git Repository') {
            steps {
                sshagent(credentials: ['Github-own-id']) {
                    script {
                        if (fileExists(env.CLONE_DIRECTORY)) {
                            echo "Repository exists. Pulling updates..."
                            dir(env.CLONE_DIRECTORY) {
                                sh """
                                    git fetch origin
                                    git checkout ${params.SOURCE_BRANCH}
                                    git reset --hard origin/${params.SOURCE_BRANCH}
                                    git pull origin ${params.SOURCE_BRANCH}
                                """
                            }
                        } else {
                            echo "Cloning repository from ${params.SOURCE_REPOSITORY_URL}..."
                            sh """
                                git clone --depth 1000 --single-branch --branch ${params.SOURCE_BRANCH} --no-tags ${params.SOURCE_REPOSITORY_URL} ${env.CLONE_DIRECTORY}
                            """
                        }
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

                        def changedFiles = changedFilesRaw ? changedFilesRaw.split("\n").findAll { it } : []
                        if (changedFiles.isEmpty()) {
                            echo "No matching files changed."
                            writeFile file: 'changed_files.txt', text: ""
                            writeFile file: 'products_files_map.txt', text: ""
                            return
                        }

                        echo "Detected ${changedFiles.size()} file(s):\n" + changedFiles.join("\n")

                        def productKeywords = params.PRODUCT_KEYWORDS
                            .split("\n")
                            .collect { it.trim().toLowerCase() }
                            .findAll { it }

                        def filteredFiles = []
                        def mapLines = []

                        for (file in changedFiles) {
                            def lowerPath = file.toLowerCase()
                            def matchedProduct = productKeywords.find { keyword ->
                                lowerPath.contains(keyword)
                            }
                            if (matchedProduct) {
                                filteredFiles.add(file)
                                mapLines.add("${matchedProduct}: ${file}")
                            }
                        }

                        if (filteredFiles) {
                            writeFile file: 'changed_files.txt', text: filteredFiles.join('\n')
                            writeFile file: 'products_files_map.txt', text: mapLines.join('\n')
                            echo "After filtering, ${filteredFiles.size()} file(s) will be deployed:\n" + filteredFiles.join("\n")
                        } else {
                            echo "No changed files matched the product keywords."
                            writeFile file: 'changed_files.txt', text: ""
                            writeFile file: 'products_files_map.txt', text: ""
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
                            for (ip in targetIPs) {
                                for (filePath in changedFiles) {
                                    def fullPath = "${env.CLONE_DIRECTORY}/${filePath}"
                                    echo "Checking existence of: ${fullPath}"
                                    if (!fileExists(fullPath)) {
                                        error "Missing file: ${fullPath}"
                                    }
                                    echo "Transferring ${filePath} to ${params.DEPLOY_USER}@${ip}:${params.DEPLOY_DIRECTORY}"
                                    sh """
                                        scp -o StrictHostKeyChecking=no '${fullPath}' '${params.DEPLOY_USER}@${ip}:${params.DEPLOY_DIRECTORY}'
                                    """
                                }
        
                                // Also transfer the product-to-file mapping file
                                def mapFilePath = "${env.CLONE_DIRECTORY}/products_files_map.txt"
                                if (fileExists(mapFilePath)) {
                                    sh """
                                        scp -o StrictHostKeyChecking=no '${mapFilePath}' '${params.DEPLOY_USER}@${ip}:${params.DEPLOY_DIRECTORY}'
                                    """
                                } else {
                                    echo "products_files_map.txt not found, skipping transfer"
                                }
                            }
                        }
                    }
                }
            }
        }

        
        stage('📧 Email Changed Files List') {
            when {
                expression {
                    fileExists("${env.CLONE_DIRECTORY}/products_files_map.txt") &&
                    readFile("${env.CLONE_DIRECTORY}/products_files_map.txt").trim()
                }
            }
            steps {
                dir(env.CLONE_DIRECTORY) {
                    script {
                        def emailBody = readFile('products_files_map.txt')
                        echo "Sending email to ${params.EMAIL_RECIPIENT} with changed files list"
                        mail bcc: '',
                             body: "Hello,\n\nHere is the list of changed files and associated products:\n\n${emailBody}\n\nRegards,\nJenkins CI",
                             cc: '',
                             replyTo: '',
                             subject: "Jenkins - Changed Files List - Build #${currentBuild.number}",
                             to: params.EMAIL_RECIPIENT
        
                        // Cleanup AFTER email is sent
                        sh 'rm -f changed_files.txt products_files_map.txt'
                    }
                }
            }
        }
    }
}
