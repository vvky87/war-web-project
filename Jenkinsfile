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
                    // Extract version from pom.xml
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
                    ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${TOMCAT_USER}@${TOMCAT_SERVER} "echo 'Connection successful'"
                """
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                script {
                    def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()
                    def warName = warFile.tokenize('/')[-1]  // Extract file name
                    def contextPath = warName.replace('.war', '')  // Remove '.war' extension

                    echo "🚀 Deploying WAR to Tomcat (Context Path: ${contextPath})..."
                    sh """
                        scp -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${warFile} ${TOMCAT_USER}@${TOMCAT_SERVER}:/tmp/
                        ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${TOMCAT_USER}@${TOMCAT_SERVER} 'sudo mv /tmp/${warName} /opt/tomcat/webapps/ && sudo systemctl restart tomcat'
                    """

                    env.APP_CONTEXT_PATH = contextPath  // Store context path for URL
                }
            }
        }

        stage('Display Application URL') {
            steps {
                echo "🌐 Deployment completed! Access your application at: http://${TOMCAT_SERVER}:8080/${env.APP_CONTEXT_PATH}"
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
