pipeline {
    agent any

    environment {
        BITBUCKET_CREDENTIALS_ID = '89f4efb3-78bb-4896-9362-601a10855e6c'
        NEXUS_CREDENTIALS_ID = 'd4295ce3-61e6-414e-b806-42d893f08c10'
        SONARQUBE_CREDENTIALS_ID = 'd2adb51e-9391-49be-8bca-9e4b41ead0e8'
        JIRA_CREDENTIALS_ID = '2a34eb6b-7cae-4013-b293-c9f0f28c587f'
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    // Checkout code from Bitbucket
                    checkout([$class: 'GitSCM', 
                              branches: [[name: '*/main']],
                              userRemoteConfigs: [[url: 'http://192.168.5.82:7990/scm/nod/nodesample.git', credentialsId: env.BITBUCKET_CREDENTIALS_ID]]])
                }
            }
        }
      
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv() {
                        // Perform SonarQube analysis
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }
         
      
        stage('Generate build.yaml') {
            steps {
                script {
                    // Define build.yaml content based on application requirements
                    def buildYamlContent = """
                    version: '3'
                    services:
                      app:
                        image: node:14
                        volumes:
                          - .:/app
                        working_dir: /app
                        command: npm start
                        ports:
                          - "3001:3001"
                        environment:
                          NODE_ENV: production
                          PORT: 3001
                    """
                    writeFile file: 'build.yaml', text: buildYamlContent
                }
            }
        }

        stage('Create Dockerfile') {
            steps {
                script {
                    // Define Dockerfile content based on application requirements
                    def dockerfileContent = """
                    FROM node:14
                    WORKDIR /usr/src/app
                    COPY package*.json ./
                    RUN npm install
                    COPY . .
                    EXPOSE 3001
                    CMD [ "npm", "start" ]
                    """
                    writeFile file: 'Dockerfile', text: dockerfileContent
                }
            }
        }
        
        stage('Build') {
            steps {
                script {
                    // Build Docker image using the Dockerfile
                    sh 'docker build -t nodejsapp:latest .'
                    
                    // Build and start services using Docker Compose
                    sh 'docker-compose -f build.yaml up -d'
                }
            }
        }
        
        stage('Publish Artifacts') {
            steps {
                script {
                    // Push Docker image to Nexus
                    def imageName = "nodejsapp"
                    def imageTag = "latest"
                    
                    // Login to Nexus Docker registry
                    withCredentials([usernamePassword(credentialsId: env.NEXUS_CREDENTIALS_ID, usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                        sh "echo \$NEXUS_PASS | docker login http://192.168.5.82:8082 --username \$NEXUS_USER --password-stdin"
                    }
                    
                    // Tag and push Docker image to Nexus
                    sh "docker tag ${imageName}:${imageTag} http://192.168.5.82:8082/NodeApp/${imageName}:${imageTag}"
                    sh "docker push http://192.168.5.82:8082/NodeApp/${imageName}:${imageTag}"
                }
            }
        }

        stage('Notify Jira') {
            steps {
                script {
                    def buildStatus = currentBuild.currentResult
                    def buildResult = (buildStatus == 'SUCCESS') ? 'Build completed successfully.' : 'Build failed.'

                    // Notify Jira about the build results
                    withCredentials([usernamePassword(credentialsId: env.JIRA_CREDENTIALS_ID, usernameVariable: 'JIRA_USER', passwordVariable: 'JIRA_PASS')]) {
                        sh """
                        curl -X POST -u \$JIRA_USER:\$JIRA_PASS \
                            -H "Content-Type: application/json" \
                            -d '{"body": "$buildResult"}' \
                            http://192.168.5.82:4000/rest/api/2/issue/DEV-4/comment
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                def buildResult = 'Build completed successfully.'
                // Notify Jira about the successful build
                withCredentials([usernamePassword(credentialsId: env.JIRA_CREDENTIALS_ID, usernameVariable: 'JIRA_USER', passwordVariable: 'JIRA_PASS')]) {
                    sh """
                    curl -X POST -u \$JIRA_USER:\$JIRA_PASS \
                        -H "Content-Type: application/json" \
                        -d '{"body": "$buildResult"}' \
                        http://192.168.5.82:4000/rest/api/2/issue/DEV-4/comment
                    """
                }
            }
        }

        failure {
            script {
                def buildResult = 'Build failed.'
                // Notify Jira about the failed build
                withCredentials([usernamePassword(credentialsId: env.JIRA_CREDENTIALS_ID, usernameVariable: 'JIRA_USER', passwordVariable: 'JIRA_PASS')]) {
                    sh """
                    curl -X POST -u \$JIRA_USER:\$JIRA_PASS \
                        -H "Content-Type: application/json" \
                        -d '{"body": "$buildResult"}' \
                        http://192.168.5.82:4000/rest/api/2/issue/DEV-4/comment
                    """
                }
            }
        }
    }
}

