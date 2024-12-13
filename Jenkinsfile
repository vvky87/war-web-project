pipeline {

    agent any

    tools {
        maven "maven"
    }

    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "3.109.133.197:8081"
        NEXUS_REPOSITORY = "demo-release"  // Using demo-release
        NEXUS_CREDENTIAL_ID = "nexus_credentials"
        ARTVERSION = "${env.BUILD_ID}"  // Using BUILD_ID for version, without SNAPSHOT
        TOMCAT_URL = "http://43.204.147.153:8080"
        TOMCAT_CREDENTIAL_ID = "tomcat_credentials"
    }

    stages {
        stage('Build WAR') {
            steps {
                // Clean and package the WAR file
                sh 'mvn clean package -DskipTests'
            }
            post {
                success {
                    echo 'Archiving artifacts...'
                    // Archive WAR file generated in the target directory
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('Publish to Nexus Repository Manager') {
            steps {
                script {
                    // Upload WAR file and POM file to Nexus repository without SNAPSHOT
                    nexusArtifactUploader(
                        nexusVersion: "${NEXUS_VERSION}",
                        protocol: "${NEXUS_PROTOCOL}",
                        nexusUrl: "${NEXUS_URL}",
                        groupId: "com.example.warwebproject",
                        version: "${ARTVERSION}",  // Using BUILD_ID for release version
                        repository: "demo-release",  // Changed to demo-release
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        artifacts: [
                            [artifactId: "war-web-project",
                             classifier: '',
                             file: "target/war-web-project-${ARTVERSION}.war",  // Updated version format
                             type: "war"],
                            [artifactId: "war-web-project",
                             classifier: '',
                             file: "pom.xml",
                             type: "pom"]
                        ]
                    )
                }
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: "${TOMCAT_CREDENTIAL_ID}", usernameVariable: 'TOMCAT_USER', passwordVariable: 'TOMCAT_PASS')]) {
                        // Undeploy existing app from Tomcat
                        sh """
                            curl -u ${TOMCAT_USER}:${TOMCAT_PASS} ${TOMCAT_URL}/manager/text/undeploy?path=/war-web-project
                        """
                        
                        // Deploy new WAR file to Tomcat
                        sh """
                            curl -u ${TOMCAT_USER}:${TOMCAT_PASS} -T target/war-web-project-${ARTVERSION}.war ${TOMCAT_URL}/manager/text/deploy?path=/war-web-project&update=true
                        """
                    }
                }
            }
        }

    }
}
