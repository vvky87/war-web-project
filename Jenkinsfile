pipeline {
    agent any
    environment {
        ART_VERSION = '1.0.0'
        TOMCAT_HOST = '43.204.112.166'
        TOMCAT_USER = 'ubuntu'
        TOMCAT_KEY = '/var/lib/jenkins/.ssh/jenkins_key'
        TOMCAT_WEBAPPS_DIR = '/opt/tomcat/webapps/'
        NEXUS_URL = 'http://3.109.203.221:8081/repository/maven-releases/'
        MAVEN_HOME = '/usr/share/maven'
        PATH = "${env.MAVEN_HOME}/bin:${env.PATH}"
    }

    stages {
        stage('Declarative: Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Verify Maven Installation') {
            steps {
                echo "🔍 Verifying Maven installation..."
                sh 'echo $PATH && mvn -v'
            }
        }

        stage('Build WAR') {
            steps {
                echo "🔨 Building WAR file..."
                sh "${env.MAVEN_HOME}/bin/mvn clean package -DskipTests"
                archiveArtifacts artifacts: 'target/*.war', allowEmptyArchive: true
            }
        }

        stage('Extract Version') {
            steps {
                script {
                    def version = sh(script: "${env.MAVEN_HOME}/bin/mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                    env.ART_VERSION = version
                    echo "📦 Detected version: ${env.ART_VERSION}"
                }
            }
        }

        stage('Publish to Nexus') {
            steps {
                echo "📦 Publishing WAR to Nexus..."
                sh """
                    find target -name *.war -print -quit | xargs -I {} ${env.MAVEN_HOME}/bin/mvn deploy:deploy-file -Dfile={} -DgroupId=com.example -DartifactId=simple-war -Dversion=${env.ART_VERSION} -DrepositoryId=maven-releases -Durl=${env.NEXUS_URL}
                """
            }
        }

        stage('Verify SSH Connection') {
            steps {
                echo "🔗 Verifying connection to Tomcat server..."
                sh """
                    ssh -i ${env.TOMCAT_KEY} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${env.TOMCAT_USER}@${env.TOMCAT_HOST} echo 'Connection successful'
                """
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                echo "🚀 Deploying WAR to Tomcat..."
                sh """
                    scp -i ${env.TOMCAT_KEY} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null target/wwp-${env.ART_VERSION}.war ${env.TOMCAT_USER}@${env.TOMCAT_HOST}:/tmp/
                    ssh -i ${env.TOMCAT_KEY} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${env.TOMCAT_USER}@${env.TOMCAT_HOST} << EOF
                        sudo mv /tmp/*.war ${env.TOMCAT_WEBAPPS_DIR}
                        sudo systemctl restart tomcat
                        sudo systemctl status tomcat
                    EOF
                """
                sleep 10
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    def deployedUrl = "http://${env.TOMCAT_HOST}:8080/wwp-${env.ART_VERSION}/"
                    def status = sh(script: "curl -s -o /dev/null -w '%{http_code}' ${deployedUrl}", returnStdout: true).trim()

                    if (status == '200') {
                        echo "✅ Deployment successful!"
                    } else {
                        error "❌ Deployment failed with status code ${status}. Check Tomcat logs!"
                    }
                }
            }
        }
    }

    post {
        success {
            echo '🎉 Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed. Check the logs above for errors.'
        }
    }
}
