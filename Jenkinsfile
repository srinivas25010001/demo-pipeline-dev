pipeline {
    agent any
    environment {
        HARBOR_CREDENTIALS = 'harbor-creds'
        DEPS_IMAGE_NAME = '10.212.132.157/dev-jute/base_image:1.0.0'
        MAIN_IMAGE_NAME = '10.212.132.157/dev-jute/devapp:latest'
        GITHUB_CREDENTIALS = 'github-creds'
        FRAPPE_DOCKER_PATH = 'frappe_docker'
        // Proxy settings
        HTTP_PROXY = 'http://192.0.2.12:8080'
        HTTPS_PROXY = 'http://192.0.2.12:8080'
        NO_PROXY = '192.0.2.50:8081'
    }
    stages {
        stage('Clone dev_jute_smart App') {
            steps {
                git branch: 'main', credentialsId: GITHUB_CREDENTIALS, url: 'https://github.com/Abhishek9938/UAT_dev_jute_smart.git'
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
                                "url": "https://${GIT_USER}:${GIT_PASS}@github.com/Abhishek9938/UAT_dev_jute_smart.git",
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
        stage('Pull Dependencies Image') {
            steps {
                sh """
                    docker pull ${DEPS_IMAGE_NAME}
                """
            }
        }
        stage('Build Main Docker Image') {
            steps {
                script {
                    def appsJsonBase64 = sh(script: "cat apps.json.b64", returnStdout: true).trim()
                    dir("${FRAPPE_DOCKER_PATH}") {
                        sh """
							docker build --no-cache \
							  --build-arg http_proxy=${HTTP_PROXY} \
							  --build-arg https_proxy=${HTTPS_PROXY} \
							  --build-arg HTTP_PROXY=${HTTP_PROXY} \
							  --build-arg HTTPS_PROXY=${HTTPS_PROXY} \
							  --build-arg no_proxy=${NO_PROXY} \
							  --build-arg NO_PROXY=${NO_PROXY} \
							  --build-arg=DEPENDENCIES_IMAGE=${DEPS_IMAGE_NAME} \
							  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
							  --build-arg=FRAPPE_BRANCH=version-15 \
							  --build-arg=PYTHON_VERSION=3.11.6 \
							  --build-arg=NODE_VERSION=18.18.2 \
							  --build-arg=APPS_JSON_BASE64='${appsJsonBase64}' \
							  --tag=${MAIN_IMAGE_NAME} \
							  --file=images/custom/Containerfile \
							  .
                        """
                    }
                }
            }
        }
        stage('Push Main Docker Image') {
            steps {
                sh "docker push ${MAIN_IMAGE_NAME}"
            }
        }
    }
    post {
        success {
            echo "Main application build and push to Harbor completed successfully!"
            echo "Dependencies Image: ${DEPS_IMAGE_NAME}"
            echo "Application Image: ${MAIN_IMAGE_NAME}"
        }
        failure {
            echo "Build or push failed."
        }
        always {
            // Clean up local images
            sh "docker rmi ${MAIN_IMAGE_NAME} || true"
            sh "docker rmi ${DEPS_IMAGE_NAME} || true"
            sh "docker builder prune -f"
        }
    }
}
