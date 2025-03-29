pipeline {
    agent any

    environment {
        TOMCAT_SERVER = "43.204.112.166"
        TOMCAT_USER = "ubuntu"
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

        stage('Extract Version') {
            steps {
                script {
                    def version = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                    env.ART_VERSION = version
                    echo "📦 Detected version: ${env.ART_VERSION}"
                }
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
                        groupId: "koddas.web.war",
                        artifactId: "wwp",
                        version: "${ART_VERSION}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        artifacts: [
                            [artifactId: "wwp", classifier: '', file: warFile, type: "war"]
                        ]
                    )
                }
            }
        }

        stage('Verify SSH Connection') {
            steps {
                echo '🔗 Verifying connection to Tomcat server...'
                sh """
                    ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${TOMCAT_USER}@${TOMCAT_SERVER} "echo 'Connection successful'"
                """
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                echo '🚀 Deploying WAR to Tomcat...'
                sh """
                    scp -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null target/*.war ${TOMCAT_USER}@${TOMCAT_SERVER}:/tmp/
                    ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${TOMCAT_USER}@${TOMCAT_SERVER} 'sudo mv /tmp/*.war /opt/tomcat/webapps/ && sudo systemctl restart tomcat'
                """
            }
        }

        stage('Display Application and Nexus URLs') {
            steps {
                script {
                    def appUrl = "http://${TOMCAT_SERVER}:8080/wwp-${env.ART_VERSION}"
                    def nexusUrl = "http://${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/koddas/web/war/wwp/${env.ART_VERSION}/wwp-${env.ART_VERSION}.war"

                    echo "<a href='${appUrl}' target='_blank'>🌐 Access Application</a>"
                    echo "<br>"
                    echo "<a href='${nexusUrl}' target='_blank'>📦 View Artifact in Nexus</a>"
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
