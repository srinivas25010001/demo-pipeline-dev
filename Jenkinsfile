pipeline {
    agent any

    environment {
        HARBOR_CREDENTIALS = 'harbor-creds'
        IMAGE_NAME = '10.212.132.157/demo/dev-demo:latest'
        GITHUB_CREDENTIALS = 'github-creds'
        FRAPPE_DOCKER_PATH = 'frappe_docker'
    }

    stages {
        stage('Clone dev_jute_smart App') {
            steps {
                git branch: 'main', credentialsId: GITHUB_CREDENTIALS, url: 'https://github.com/srinivas25010001/dev_jute_smart.git'
            }
        }

        stage('Create apps.json') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: GITHUB_CREDENTIALS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                        def appsJsonContent = """
                        [
                            {
                                "url": "https://github.com/frappe/erpnext",
                                "branch": "version-15"
                            },
                            {
                                "url": "https://${GIT_USER}:${GIT_PASS}@github.com/srinivas25010001/dev_jute_smart.git",
                                "branch": "main"
                            }
                        ]
                        """
                        writeFile(file: 'apps.json', text: appsJsonContent)
                    }
                }
            }
        }

        stage('Encode apps.json') {
            steps {
                sh 'base64 -w 0 apps.json > apps.json.b64'
            }
        }

        stage('Login to Harbor') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: HARBOR_CREDENTIALS, usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                        sh """
                            echo "$HARBOR_PASS" | docker login 10.212.132.157 -u "$HARBOR_USER" --password-stdin
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
    steps {
        script {
            def appsJsonBase64 = sh(script: "cat apps.json.b64", returnStdout: true).trim()
            dir("${FRAPPE_DOCKER_PATH}") {
                sh """
                    docker build \
                      --build-arg HTTP_PROXY=http://192.0.2.12:8080 \
                      --build-arg HTTPS_PROXY=http://192.0.2.12:8080 \
                      --build-arg NO_PROXY=192.0.2.50:8081 \
                      --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
                      --build-arg=FRAPPE_BRANCH=version-15 \
                      --build-arg=PYTHON_VERSION=3.11.6 \
                      --build-arg=NODE_VERSION=18.18.2 \
                      --build-arg=APPS_JSON_BASE64='${appsJsonBase64}' \
                      --tag=${IMAGE_NAME} \
                      --file=images/custom/Containerfile \
                      .
                """
            }
        }
    }
}


        stage('Push Docker Image') {
            steps {
                sh "docker push ${IMAGE_NAME}"
            }
        }
    }

    post {
        success {
            echo "Build and push to Harbor completed successfully!"
        }
        failure {
            echo "Build or push failed."
        }
        always {
            // Clean up local image
            sh "docker rmi ${IMAGE_NAME} || true"
            sh "docker builder prune -f"
        }
    }
}
