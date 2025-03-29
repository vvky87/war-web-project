pipeline {
    agent any  // Runs on any available Jenkins agent

    environment {
        TOMCAT_SERVER = "43.204.112.166"
        TOMCAT_USER = "ubuntu"
        TOMCAT_CREDENTIAL_ID = "tomcat_creds"
        ART_VERSION = "1.0.0"
        NEXUS_URL = "3.109.203.221:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"
    }

    tools {
        maven "maven"  // Using Maven for building the project
    }

    stages {
        stage('Build WAR') {
            steps {
                echo 'Building WAR file...'
                sh 'mvn clean package -DskipTests'  // Build the project without running tests
                archiveArtifacts artifacts: '**/target/*.war'  // Archive the WAR file
            }
        }

        stage('Publish to Nexus') {
            steps {
                echo 'Publishing WAR to Nexus...'
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

        stage('Deploy to Tomcat') {
            steps {
                echo 'Deploying WAR to Tomcat...'
                withCredentials([sshUserPrivateKey(credentialsId: TOMCAT_CREDENTIAL_ID, keyFileVariable: 'SSH_KEY')]) {
                    sh '''
                        scp -i ${SSH_KEY} -o StrictHostKeyChecking=no target/*.war ${TOMCAT_USER}@${TOMCAT_SERVER}:/tmp/
                        ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${TOMCAT_USER}@${TOMCAT_SERVER} << 'EOF'
                            sudo mv /tmp/*.war /opt/tomcat/webapps/
                            sudo systemctl restart tomcat
                        EOF
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Deployment successful!'
        }
        failure {
            echo '❌ Deployment failed. Check the logs!'
        }
    }
}
