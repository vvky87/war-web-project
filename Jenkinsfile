pipeline {
    agent any

    environment {
        TOMCAT_SERVER = "43.204.112.166"
        TOMCAT_USER = "ubuntu" // Changed from root to ubuntu
        TOMCAT_URL = "http://${TOMCAT_SERVER}:8080"
        ART_VERSION = "1.0.0"
        NEXUS_URL = "3.109.203.221:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"
        TOMCAT_CREDENTIAL_ID = "tomcat_creds" // Credentials for copying WAR
    }

    tools {
        maven "maven"
    }

    stages {
        stage('Build WAR') {
            steps {
                sh '''
                    #!/bin/bash
                    mvn clean package -DskipTests
                '''
                archiveArtifacts artifacts: '**/target/*.war'
            }
        }

        stage('Publish to Nexus') {
            steps {
                script {
                    def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()
                    def pomFile = "pom.xml"

                    if (!fileExists(warFile)) {
                        error("WAR file not found at ${warFile}")
                    }
                    if (!fileExists(pomFile)) {
                        error("POM file not found.")
                    }

                    nexusArtifactUploader(
                        nexusVersion: "nexus3",
                        protocol: "http",
                        nexusUrl: "${NEXUS_URL}",
                        groupId: "com.example.warwebproject",
                        version: "${ART_VERSION}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        artifacts: [
                            [artifactId: "wwp", classifier: '', file: warFile, type: "war"],
                            [artifactId: "wwp", classifier: '', file: pomFile, type: "pom"]
                        ]
                    )
                }
            }
        }

        stage('Deploy WAR') {
            steps {
                script {
                    def warFilePath = sh(script: "find target -name '*.war' -type f -print0 | xargs -0 ls -t | head -n 1", returnStdout: true).trim()

                    if (warFilePath) {
                        echo "WAR file located at: ${warFilePath}"

                        sshagent([TOMCAT_CREDENTIAL_ID]) {
                            sh '''
                                #!/bin/bash
                                set -e

                                # Copy WAR file to the remote server
                                scp -o StrictHostKeyChecking=no -o BatchMode=yes ${warFilePath} ${TOMCAT_USER}@${TOMCAT_SERVER}:/tmp/wwp.war || { echo "SCP failed."; exit 1; }

                                # Deploy WAR file on the remote server
                                ssh -o StrictHostKeyChecking=no -o BatchMode=yes ${TOMCAT_USER}@${TOMCAT_SERVER} <<'EOF'
                                    set -e
                                    echo "Moving WAR file to Tomcat directory..."
                                    sudo mv /tmp/wwp.war /opt/tomcat/webapps/wwp.war
                                    echo "WAR file deployed successfully."
                                    echo "Restarting Tomcat..."
                                    sudo systemctl restart tomcat
                                    echo "Tomcat restarted."
                                EOF
                            '''
                        }
                    } else {
                        error "No WAR file found to deploy."
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed.'
        }
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed. Check logs for details.'
        }
    }
}
