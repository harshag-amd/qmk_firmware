pipeline {
    agent any

    parameters {
        string(name: 'USERNAME', defaultValue: 'herky', description: 'Username for target server(s)')
        text(name: 'TARGET_HOSTS', defaultValue: '10.86.22.146', description: 'List of target server IPs (one per line)')
        string(name: 'TARGET_PATH', defaultValue: '/opt/firmware/', description: 'Destination path on target servers')
        text(name: 'FIRMWARE_FILES', defaultValue: 'harsha\nherk', description: 'List of firmware files to monitor (one per line)')
        choice(name: 'PKG_MANAGER', choices: ['apt', 'dnf', 'yum'], description: 'Package manager to run system update')
    }

    environment {
        FIRMWARE_DIR_REPO = "firmware"
    }

    triggers {
        pollSCM('* * * * *') // Every minute
    }

    stages {
        stage('Declarative: Pipeline Intializing') {
            steps {
                echo 'Pipeline initialized.'
            }
        }

        stage('Detect Changes') {
            steps {
                script {
                    def changeLog = currentBuild.changeSets
                    def changedFiles = [] as Set
        
                    for (change in changeLog) {
                        for (file in change.items.collectMany { it.affectedFiles }) {
                            changedFiles << file.path
                        }
                    }
        
                    env.CHANGED_FILES = changedFiles.join(",")
                    if (changedFiles.isEmpty()) {
                        echo "No files changed."
                    } else {
                        echo "Detected changes in: ${env.CHANGED_FILES}"
                    }
                }
            }
        }


		stage('Copy Changed Files') {
		    when {
		        expression { return env.CHANGED_FILES?.trim() }
		    }
		    steps {
		        sshagent(credentials: ['your-jenkins-ssh-credential-id']) {
		            script {
		                def hosts = params.TARGET_HOSTS.split("\n").collect { it.trim() }.findAll { it }
		                def changedFiles = env.CHANGED_FILES.split(",").collect { it.trim() }
		
		                for (host in hosts) {
		                    for (file in changedFiles) {
		                        if (fileExists(file)) {
		                            def filename = file.tokenize("/").last()
		                            sh """
		                                echo "Copying ${file} to ${host}..."
		                                scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${file} ${params.USERNAME}@${host}:${params.TARGET_PATH}/${filename}
		                            """
		                        } else {
		                            echo "Warning: ${file} not found in workspace."
		                        }
		                    }
		                }
		            }
		        }
		    }
		}


        stage('Update Servers') {
            when {
                expression { return env.CHANGED_FILES?.trim() }
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
                            sh """
                                echo "Running ${params.PKG_MANAGER} update on ${host}..."
                                echo 'w' | ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${params.USERNAME}@${host} '${updateCmd}'
                            """
                        }
                    }
                }
            }
        }

        stage('No Change Detected') {
            when {
                not {
                    expression { return env.CHANGED_FILES?.trim() }
                }
            }
            steps {
                echo "No firmware changes detected. Skipping copy and update stages."
            }
        }
    }
}
