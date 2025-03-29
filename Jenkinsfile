pipeline {
    agent any

    environment {
        TOMCAT_SERVER = "43.204.112.166"
        TOMCAT_USER = "ubuntu"
        ART_VERSION = "1.0.0"
        NEXUS_URL = "3.109.203.221:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"
        SSH_KEY_PATH = "/var/lib/jenkins/.ssh/jenkins_key"
    }

    tools {
        maven "maven"
    }

    stages {
        stage('Build WAR') {
            steps {
                echo '🔨 Building WAR file...'
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: '**/target/*.war'
            }
        }

        stage('Publish to Nexus') {
            steps {
                echo '📦 Publishing WAR to Nexus...'
                script {
                    def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()
                    nexusArtifactUploader(
                        nexusVersion: "nexus3",
                        protocol: "http",
                        nexusUrl: "${NEXUS_URL}",
                        groupId: "com.example",
                        version: "${ART_VERSION}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        artifacts: [
                            [artifactId: "simple-war", classifier: '', file: warFile, type: "war"]
                        ]
                    )
                }
            }
        }

        stage('Verify SSH Connection') {
            steps {
                echo '🔗 Verifying connection to Tomcat server...'
                sh """
                    ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${TOMCAT_USER}@${TOMCAT_SERVER} 'echo "Connection successful"'
                """
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                echo '🚀 Deploying WAR to Tomcat...'
                script {
                    def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()
                    if (fileExists(warFile)) {
                        sh """
                            scp -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${warFile} ${TOMCAT_USER}@${TOMCAT_SERVER}:/tmp/
                            ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${TOMCAT_USER}@${TOMCAT_SERVER} 'sudo mv /tmp/$(basename ${warFile}) /opt/tomcat/webapps/ && sudo systemctl restart tomcat'
                        """
                    } else {
                        error "❌ WAR file not found: ${warFile}"
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                echo '✅ Verifying deployment...'
                script {
                    def status = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://${TOMCAT_SERVER}:8080/simple-war", returnStdout: true).trim()
                    if (status == "200") {
                        echo "🎉 Application deployed successfully and is accessible!"
                    } else {
                        error "❌ Deployment verification failed with HTTP code ${status}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed. Check the logs for errors.'
        }
    }
}
