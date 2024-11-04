pipeline {
    agent any
    environment {
        IMAGE = "neathtan/nextjs-adv"
        FILE_NAME = "React.zip"
        DIR_UNZIP = "React"
        DOCKER_IMAGE = "${IMAGE}:${BUILD_NUMBER}"
        DOCKER_CONTAINER = "Nextjs_jenkins"
        DOCKER_CREDENTIALS_ID = "dockertoken"
        GIT_BRANCH = "master"
        GIT_MANIFEST_REPO = "https://github.com/WexleyTan/auto_nextjs_manifest.git"
        MANIFEST_REPO = "auto_nextjs_manifest"
        MANIFEST_FILE_PATH = "deployment.yaml"
        GIT_CREDENTIALS_ID = 'git_pass'
    }

    stages {
        stage('Unzip File') {
            steps {
                script {
                    echo "Checking if the file ${FILE_NAME} exists and unzipping it if present..."
                    sh """
                        if [ -f '${FILE_NAME}' ]; then
                            echo "Removing existing directory ${DIR_UNZIP}..."
                            rm -rf ${DIR_UNZIP}
                            echo "Unzipping the file..."
                            unzip -o '${FILE_NAME}'
                        else
                            echo "'${FILE_NAME}' does not exist."
                            exit 1
                        fi
                    """
                }
            }
        }

        stage('Create Dockerfile') {
            steps {
                script {
                    dir ("${DIR_UNZIP}") {
                        echo "Creating Dockerfile..."
                        writeFile file: 'Dockerfile', text: '''
                        FROM node:18-alpine

                        WORKDIR /app
                        
                        COPY . .
                        
                        RUN yarn install
                        
                        COPY . .
                        
                        EXPOSE 5173
                        
                        CMD [ "yarn", "run", "dev" ]
                        '''
                    }
                }
            }
        }

        stage("Build and Push Docker Image") {
            steps {
                script {
                    echo "Building Docker image..."
                    dir("${DIR_UNZIP}") { 
                        sh "docker build -t ${DOCKER_IMAGE} ."  
                    }
                    sh "docker images | grep -i ${IMAGE}"

                    echo "Logging in to Docker Hub using Jenkins credentials..."
                    withCredentials([usernamePassword(credentialsId: DOCKER_CREDENTIALS_ID, passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    }
                    
                    echo "Pushing the image to Docker Hub..."
                    sh "docker push ${DOCKER_IMAGE}"
                }
            }
        }

        stage("Cloning the Manifest File") {
            steps {
                script {
                    echo "Checking if the manifest repository exists and removing it if necessary..."
                    sh """
                        if [ -d "${MANIFEST_REPO}" ]; then
                            echo "Directory ${MANIFEST_REPO} exists, removing it..."
                            rm -rf ${MANIFEST_REPO}
                        fi
                    """
                    
                    echo "Cloning the manifest repository..."
                    sh "git clone -b ${GIT_BRANCH} ${GIT_MANIFEST_REPO} ${MANIFEST_REPO}"  
                }
            }
        }
        
        stage("Updating the Manifest File") {
            steps {
                script {
                    echo "Updating the image in the deployment manifest..."
                    dir("${MANIFEST_REPO}") {
                        sh """
                            sed -i 's|image: ${IMAGE}:.*|image: ${DOCKER_IMAGE}|' ${MANIFEST_FILE_PATH}
                            echo "Updated deployment file:"
                        """
                        
                        echo "Committing and pushing changes to the manifest repository..."
                        withCredentials([usernamePassword(credentialsId: GIT_CREDENTIALS_ID, passwordVariable: 'GIT_PASS', usernameVariable: 'GIT_USER')]) {
                            sh """
                                git config --global user.name "WexleyTan"
                                git config --global user.email "neathtan1402@gmail.com"
                                git add ${MANIFEST_FILE_PATH}
                                git commit -m "Update image to ${DOCKER_IMAGE}"
                                git push https://${GIT_USER}:${GIT_PASS}@github.com/WexleyTan/auto_nextjs_manifest ${GIT_BRANCH}
                            """
                        }
                    }
                }
            }
        }
    }
}
