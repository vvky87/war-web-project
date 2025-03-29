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
                    def groupId = sh(script: "mvn help:evaluate -Dexpression=project.groupId -q -DforceStdout", returnStdout: true).trim()
                    def artifactId = sh(script: "mvn help:evaluate -Dexpression=project.artifactId -q -DforceStdout", returnStdout: true).trim()
                    def version = env.ART_VERSION

                    echo "Publishing ${artifactId}-${version}.war with groupId: ${groupId}"

                    nexusArtifactUploader(
                        nexusVersion: "nexus3",
                        protocol: "http",
                        nexusUrl: "${NEXUS_URL}",
                        groupId: "${groupId}",
                        version: "${version}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        artifacts: [
                            [artifactId: "${artifactId}", classifier: '', file: warFile, type: "war"]
                        ]
                    )

                    // Display Nexus URL after upload
                    def nexusUrl = "http://${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/${groupId.replace('.', '/')}/${artifactId}/${version}/${artifactId}-${version}.war"
                    env.NEXUS_URL_DISPLAY = nexusUrl
                }
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                echo '🚀 Deploying WAR to Tomcat...'
                script {
                    def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()
                    def artifactId = sh(script: "mvn help:evaluate -Dexpression=project.artifactId -q -DforceStdout", returnStdout: true).trim()

                    sh """
                        scp -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${warFile} ${TOMCAT_USER}@${TOMCAT_SERVER}:/tmp/
                        ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${TOMCAT_USER}@${TOMCAT_SERVER} 'sudo mv /tmp/${artifactId}-*.war /opt/tomcat/webapps/ && sudo systemctl restart tomcat'
                    """
                }
            }
        }

        stage('Display Application and Nexus URLs') {
            steps {
                script {
                    def artifactId = sh(script: "mvn help:evaluate -Dexpression=project.artifactId -q -DforceStdout", returnStdout: true).trim()
                    def appUrl = "http://${TOMCAT_SERVER}:8080/${artifactId}-${env.ART_VERSION}"

                    echo "🌐 Deployment completed! Access your application at: ${appUrl}"
                    echo "📦 Artifact available at Nexus: ${env.NEXUS_URL_DISPLAY}"
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
