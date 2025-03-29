pipeline {
    agent any

    environment {
        ART_VERSION = "1.0.0"
        NEXUS_URL = "3.109.203.221:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"
        TOMCAT_SERVER = "43.204.112.166"
        TOMCAT_USER = "ubuntu"  // For connecting to Tomcat server
        TOMCAT_CREDENTIAL_ID = "tomcat_creds"  // For status check post-deployment
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
                script {
                    def sshCommand = """
                        ssh -o BatchMode=yes -o ConnectTimeout=5 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${TOMCAT_USER}@${TOMCAT_SERVER} 'echo connection successful'
                    """
                    def sshTest = sh(script: sshCommand, returnStatus: true)

                    if (sshTest != 0) {
                        error "❌ Unable to establish SSH connection to Tomcat server at ${TOMCAT_SERVER}."
                    } else {
                        echo "✅ SSH connection established successfully!"
                    }
                }
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                echo '🚀 Deploying WAR to Tomcat...'
                sh """
                    scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null target/*.war ${TOMCAT_USER}@${TOMCAT_SERVER}:/tmp/
                    ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${TOMCAT_USER}@${TOMCAT_SERVER} << 'EOF'
                        sudo mv /tmp/*.war /opt/tomcat/webapps/
                        sudo systemctl restart tomcat
                    EOF
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                echo '✅ Verifying deployment with Tomcat credentials...'
                withCredentials([usernamePassword(credentialsId: "${TOMCAT_CREDENTIAL_ID}", usernameVariable: 'TOMCAT_USER', passwordVariable: 'TOMCAT_PASSWORD')]) {
                    script {
                        def status = sh(script: """
                            curl -u ${TOMCAT_USER}:${TOMCAT_PASSWORD} -s -o /dev/null -w '%{http_code}' http://${TOMCAT_SERVER}:8080/simple-war
                        """, returnStdout: true).trim()

                        if (status == "200") {
                            echo "🎉 Application deployed successfully and is accessible!"
                        } else {
                            error "❌ Deployment verification failed with HTTP code ${status}"
                        }
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
