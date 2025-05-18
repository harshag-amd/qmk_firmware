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
strix''', description: 'Keywords to filter relevant products (one per line)')
        string(name: 'EMAIL_RECIPIENT', defaultValue: 'harshavardhanreddy.gangavaram@amd.com', description: 'Email to send changed files list')
    }

    environment {
        TRACKING_COMMIT_FILE = "${WORKSPACE}/.last_successful_commit"
    }

    stages {
        stage('🧪 Validate Inputs') {
            steps {
                script {
                    if (!params.ALLOWED_FILE_EXTENSIONS?.trim()) error "ALLOWED_FILE_EXTENSIONS must not be empty!"
                    if (!params.TARGET_SERVER_IPS?.trim()) error "TARGET_SERVER_IPS must not be empty!"
                }
            }
        }

        stage('🔍 Detect Changes') {
            steps {
                script {
                    def lastCommit = fileExists(env.TRACKING_COMMIT_FILE) ? readFile(env.TRACKING_COMMIT_FILE).trim() : ''
                    def currentCommit = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                    def diffRange = lastCommit ? "${lastCommit}..${currentCommit}" : "HEAD~1..HEAD"

                    echo "Comparing commits: ${diffRange}"

                    // Normalize extensions
                    def extensions = []
                    for (ext in params.ALLOWED_FILE_EXTENSIONS.split('\n')) {
                        ext = ext.trim()
                        if (ext) {
                            extensions << ext.replaceFirst('^\\.', '')
                        }
                    }
                    def extRegex = extensions.join('|')

                    // Get changed files
                    def changedFilesRaw = sh(
                        script: "git diff --name-only ${diffRange} | grep -Ei '\\.(${extRegex})\$' || true",
                        returnStdout: true
                    ).trim()
                    def changedFiles = changedFilesRaw ? changedFilesRaw.split('\n') : []
                    echo "Changed files:\n${changedFiles.join('\n')}"

                    // Normalize product keywords
                    def productKeywords = []
                    for (kw in params.PRODUCT_KEYWORDS.split('\n')) {
                        kw = kw.trim()
                        if (kw) {
                            productKeywords << kw.toLowerCase()
                        }
                    }

                    // Filter by product keywords
                    def filteredFiles = []
                    def mapLines = []

                    for (f in changedFiles) {
                        def lower = f.toLowerCase()
                        def matched = productKeywords.find { kw -> lower.contains(kw) }
                        if (matched) {
                            filteredFiles << f
                            mapLines << "${matched}: ${f}"
                        }
                    }

                    if (filteredFiles) {
                        writeFile file: 'changed_files.txt', text: filteredFiles.join('\n')
                        writeFile file: 'products_files_map.txt', text: mapLines.join('\n')
                        echo "Filtered files:\n${filteredFiles.join('\n')}"
                    } else {
                        writeFile file: 'changed_files.txt', text: ''
                        writeFile file: 'products_files_map.txt', text: ''
                        echo "No matching product files found."
                    }

                    writeFile file: env.TRACKING_COMMIT_FILE, text: currentCommit
                }
            }
        }

        stage('🚀 Deploy to Targets') {
            when {
                expression {
                    fileExists('changed_files.txt') && readFile('changed_files.txt').trim()
                }
            }
            steps {
                script {
                    def changedFiles = readFile('changed_files.txt').split('\n').findAll { it?.trim() }
                    def targetIPs = params.TARGET_SERVER_IPS.split('\n').findAll { it?.trim() }

                    sshagent(credentials: ['your-jenkins-ssh-credential-id']) {
                        for (ip in targetIPs) {
                            for (filePath in changedFiles) {
                                if (!fileExists(filePath)) {
                                    error "File not found: ${filePath}"
                                }
                                echo "Transferring ${filePath} to ${params.DEPLOY_USER}@${ip}:${params.DEPLOY_DIRECTORY}"
                                sh """
                                    scp -o StrictHostKeyChecking=no '${filePath}' '${params.DEPLOY_USER}@${ip}:${params.DEPLOY_DIRECTORY}'
                                """
                            }

                            if (fileExists('products_files_map.txt')) {
                                sh """
                                    scp -o StrictHostKeyChecking=no 'products_files_map.txt' '${params.DEPLOY_USER}@${ip}:${params.DEPLOY_DIRECTORY}'
                                """
                            }
                        }
                    }
                }
            }
        }

        stage('📧 Email Report') {
            when {
                expression {
                    fileExists('products_files_map.txt') && readFile('products_files_map.txt').trim()
                }
            }
            steps {
                script {
                    def emailBody = readFile('products_files_map.txt')
                    echo "Sending email to ${params.EMAIL_RECIPIENT}"
                    mail bcc: '',
                         body: "Hello,\n\nHere is the list of changed files and associated products:\n\n${emailBody}\n\nRegards,\nJenkins CI",
                         cc: '',
                         replyTo: '',
                         subject: "Jenkins - Changed Files List - Build #${currentBuild.number}",
                         to: params.EMAIL_RECIPIENT

                    sh 'rm -f changed_files.txt products_files_map.txt'
                }
            }
        }
    }
}
