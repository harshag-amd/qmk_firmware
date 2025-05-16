pipeline {
    agent any

    environment {
        REPO_DIR_PATH = "${WORKSPACE}/${params.REPO_DIR}"
    }

    parameters {
        string(name: 'USERNAME', defaultValue: 'herky', description: '')
        text(name: 'TARGET_HOSTS', defaultValue: '10.86.22.146', description: '')
        string(name: 'TARGET_PATH', defaultValue: '/opt/firmware/', description: '')
        text(name: 'ALLOWED_EXTENSIONS', defaultValue: '.h\n.c\n.cpp\n.bin', description: '')
        string(name: 'GIT_REPO_URL', defaultValue: 'https://github.com/harshag-amd/qmk_firmware.git', description: '')
        string(name: 'GIT_BRANCH', defaultValue: 'master', description: '')
        string(name: 'REPO_DIR', defaultValue: '.', description: '')
        text(name: 'PRODUCT_LIST', defaultValue: 'arden\nnavi\nkracken\nraphael\nremambrant', description: '')
    }

    stages {
       



        stage('Detect Changes') {
            steps {
                dir("${params.REPO_DIR}") {
                    script {
                        def latestCommit = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                        echo "Latest Commit: ${latestCommit}"

                        def changedFilesStr = sh(script: "git diff-tree --no-commit-id --name-only -r ${latestCommit}", returnStdout: true).trim()
                        echo "Changed files:\n${changedFilesStr}"

                        def changedFiles = changedFilesStr.split("\n").findAll { it }
                        def extensions = params.ALLOWED_EXTENSIONS.split("\n").collect { it.trim() }
                        def products = params.PRODUCT_LIST.split("\n").collect { it.trim().toLowerCase() }

                        def matchedFiles = [] as Set
                        def detectedProducts = [] as Set

                        for (file in changedFiles) {
                            if (!extensions.any { ext -> file.endsWith(ext) }) continue
                            def matchedProduct = products.find { product -> file.toLowerCase().contains(product) }
                            if (matchedProduct) {
                                matchedFiles << file
                                detectedProducts << matchedProduct
                            }
                        }

                        matchedFilesGlobal = matchedFiles as List
                        detectedProductsGlobal = detectedProducts as List

                        echo "Matched files:\n${matchedFilesGlobal.join('\n')}"
                        echo "Detected products: ${detectedProductsGlobal.join(', ')}"
                    }
                }
            }
        }

        stage('Copy Files') {
            when {
                expression { return matchedFilesGlobal && matchedFilesGlobal.size() > 0 }
            }
            steps {
                dir("${params.REPO_DIR}") {
                    script {
                        def hosts = params.TARGET_HOSTS.split("\n").findAll { it }
                        for (host in hosts) {
                            for (file in matchedFilesGlobal) {
                                if (fileExists(file)) {
                                    def fname = file.tokenize("/")[-1]
                                    echo "Copying ${file} to ${host}:${params.TARGET_PATH}/${fname}"
                                    sh "scp -o StrictHostKeyChecking=no ${file} ${params.USERNAME}@${host}:${params.TARGET_PATH}/${fname}"
                                } else {
                                    echo "File ${file} not found"
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('No Changes') {
            when {
                expression { return matchedFilesGlobal.isEmpty() }
            }
            steps {
                echo "No matching files found, skipping."
            }
        }
    }
}
