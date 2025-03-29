pipeline {
    agent any

    environment {
        TOMCAT_SERVER = "43.204.112.166"
        TOMCAT_USER = "ubuntu"
        NEXUS_URL = "3.109.203.221:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"
        SSH_KEY_PATH = "/var/lib/jenkins/.ssh/jenkins_key"
        ART_VERSION = "0.0.0"  // Default version
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
                    def version = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                    
                    // Debugging the version output
                    echo "🔍 Extracted Version: ${version}"
                    
                    if (version) {
                        env.ART_VERSION = version
                    } else {
                        error "❌ Version extraction failed. Check the pom.xml file."
                    }

                    nexusArtifactUploader(
                        nexusVersion: "nexus3",
                        protocol: "http",
                        nexusUrl: "${NEXUS_URL}",
                        groupId: "koddas.web.war",
                        artifactId: "wwp",
                        version: "${env.ART_VERSION}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        artifacts: [[artifactId: "wwp", file: warFile, type: "war"]]
                    )
                    
                    def nexusUrl = "http://${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/koddas/web/war/wwp/${env.ART_VERSION}/wwp-${env.ART_VERSION}.war"
                    echo "📦 Nexus Artifact URL: ${nexusUrl}"
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
                    scp -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no target/*.war ${TOMCAT_USER}@${TOMCAT_SERVER}:/tmp/
                    ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ${TOMCAT_USER}@${TOMCAT_SERVER} 'sudo mv /tmp/*.war /opt/tomcat/webapps/ && sudo systemctl restart tomcat'
                """
                script {
                    def appUrl = "http://${TOMCAT_SERVER}:8080/wwp-${env.ART_VERSION}"
                    echo "🚀 Application URL after deployment: ${appUrl}"
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
