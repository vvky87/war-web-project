pipeline {
    agent any  // Runs on any available Jenkins agent

    environment {
        TOMCAT_SERVER = "43.204.112.166"           // IP of the Tomcat server
        TOMCAT_USER = "ubuntu"                    // SSH username for Tomcat
        TOMCAT_CREDENTIAL_ID = "tomcat_creds"     // Jenkins credentials ID for SSH (private key or username/password)
        ART_VERSION = "1.0.0"                     // Version of the WAR file
        NEXUS_URL = "3.109.203.221:8081"          // Nexus repository URL
        NEXUS_REPOSITORY = "maven-releases"       // Nexus repository name
        NEXUS_CREDENTIAL_ID = "nexus_creds"       // Jenkins credentials for Nexus
    }

    tools {
        maven "maven"  // Use Maven to build the project
    }

    stages {
        // Stage 1: Build the WAR file
        stage('Build WAR') {
            steps {
                echo '🔨 Building WAR file...'
                sh 'mvn clean package -DskipTests'  // Build without running tests
                archiveArtifacts artifacts: '**/target/*.war'  // Archive the WAR file for later use
            }
        }

        // Stage 2: Publish the WAR to Nexus Repository
        stage('Publish to Nexus') {
            steps {
                echo '📦 Publishing WAR to Nexus...'
                script {
                    def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()

                    nexusArtifactUploader(
                        nexusVersion: "nexus3",
                        protocol: "http",
                        nexusUrl: "${NEXUS_URL}",
                        groupId: "com.example",   // Group ID from pom.xml
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

        // Stage 3: Deploy the WAR to Tomcat
    stage('Deploy to Tomcat') {
    steps {
        echo '🚀 Deploying WAR to Tomcat...'
        sshagent (credentials: ['tomcat_creds']) {
            sh '''
                scp -o StrictHostKeyChecking=no target/*.war ${TOMCAT_USER}@${TOMCAT_SERVER}:/tmp/
                ssh -o StrictHostKeyChecking=no ${TOMCAT_USER}@${TOMCAT_SERVER} << 'EOF'
                    sudo mv /tmp/*.war /opt/tomcat/webapps/
                    sudo systemctl restart tomcat
                EOF
            '''
        }
    }
}


        // Stage 4: Verify Deployment
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

    // Post actions
    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed. Check the logs for errors.'
        }
    }
}
