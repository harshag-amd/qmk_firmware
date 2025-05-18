pipeline {
    agent any

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

    environment {
        TRACKING_COMMIT_FILE = "${WORKSPACE}/repo/.last_successful_commit"
        CLONE_DIRECTORY = "${WORKSPACE}/repo"
    }

    triggers {
        pollSCM('* * * * *')
    }

    stages {
        stage('🧪 Validate Input Parameters') {
            steps {
                script {
                    if (!params.ALLOWED_FILE_EXTENSIONS?.trim()) error "ALLOWED_FILE_EXTENSIONS must not be empty!"
                    if (!params.TARGET_SERVER_IPS?.trim()) error "TARGET_SERVER_IPS must not be empty!"
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
                            echo "Cloning repository..."
                            sh "git clone --depth 1000 --branch ${params.SOURCE_BRANCH} ${params.SOURCE_REPOSITORY_URL} ${env.CLONE_DIRECTORY}"
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
                        echo "Checking changes between ${diffRange}"

                        def extPattern = params.ALLOWED_FILE_EXTENSIONS
                            .split("\n").collect { it.trim().replaceAll("^\\.", "") }.findAll { it }
                            .join("|")
                        if (!extPattern) error "No valid file extensions found"

                        def changedRaw = sh(
                            script: "git diff --name-only ${diffRange} | grep -Ei '\\.(${extPattern})\$' || true",
                            returnStdout: true
                        ).trim()

                        def changedFiles = changedRaw ? changedRaw.split("\n").findAll { it } : []
                        def productKeywords = params.PRODUCT_KEYWORDS.split("\n").collect { it.trim().toLowerCase() }.findAll { it }

                        def filtered = []
                        def mapping = []

                        changedFiles.each { file ->
                            def lower = file.toLowerCase()
                            def match = productKeywords.find { lower.contains(it) }
                            if (match) {
                                filtered << file
                                mapping << "${match}: ${file}"
                            }
                        }

                        writeFile file: 'changed_files.txt', text: filtered.join("\n")
                        writeFile file: 'products_files_map.txt', text: mapping.join("\n")
                        writeFile file: env.TRACKING_COMMIT_FILE, text: currentCommit

                        echo filtered ? "Detected ${filtered.size()} changed file(s):\n${filtered.join('\n')}" :
                                        "No changed files matched product keywords"
                    }
                }
            }
        }

        stage('🚀 Deploy Modified Files') {
            steps {
                script {
                    if (!fileExists('changed_files.txt')) {
                        echo "changed_files.txt does not exist. Skipping deployment."
                        return
                    }

                    def changed = readFile('changed_files.txt').trim()
                    if (!changed) {
                        echo "No files to deploy."
                        return
                    }

                    def files = changed.split("\n").findAll { it }
                    def ips = params.TARGET_SERVER_IPS.split("\n").findAll { it }

                    sshagent(credentials: ['your-jenkins-ssh-credential-id']) {
                        for (ip in ips) {
                            for (file in files) {
                                def full = "${env.CLONE_DIRECTORY}/${file}"
                                if (!fileExists(full)) {
                                    error "File not found: ${full}"
                                }
                                echo "Deploying ${file} to ${params.DEPLOY_USER}@${ip}:${params.DEPLOY_DIRECTORY}"
                                sh """
                                    scp -o StrictHostKeyChecking=no '${full}' '${params.DEPLOY_USER}@${ip}:${params.DEPLOY_DIRECTORY}'
                                """
                            }

                            def mapFile = "${env.CLONE_DIRECTORY}/products_files_map.txt"
                            if (fileExists(mapFile)) {
                                sh "scp -o StrictHostKeyChecking=no '${mapFile}' '${params.DEPLOY_USER}@${ip}:${params.DEPLOY_DIRECTORY}'"
                            } else {
                                echo "Mapping file not found; skipping"
                            }
                        }
                    }
                }
            }
        }

        stage('📧 Email Changed Files List') {
            steps {
                script {
                    def mapFile = "${env.CLONE_DIRECTORY}/products_files_map.txt"
                    if (!fileExists(mapFile) || !readFile(mapFile).trim()) {
                        echo "No product-to-file mappings found. Skipping email."
                        return
                    }

                    def emailBody = readFile(mapFile)
                    mail bcc: '',
                         body: "Hello,\n\nHere is the list of changed files and associated products:\n\n${emailBody}\n\nRegards,\nJenkins CI",
                         cc: '',
                         replyTo: '',
                         subject: "Jenkins - Changed Files List - Build #${currentBuild.number}",
                         to: params.EMAIL_RECIPIENT

                    sh "rm -f changed_files.txt products_files_map.txt"
                }
            }
        }
    }
}
