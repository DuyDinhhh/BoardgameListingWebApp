pipeline {
    agent { label 'JDK17' }

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        NEXUS_URL = '192.168.64.37:8081'
        NEXUS_CREDENTIALS_ID = 'for-nexus'
        NEXUS_REPOSITORY = 'maven-releases'
        NEXUS_GROUP = 'com/javaproject'
        NEXUS_ARTIFACT_ID = 'database_service_project'
        ARTIFACT_VERS="1.${env.BUILD_ID}"
    }

    stages {
        stage('Branch Validation') {
            steps {
                script {
                    if (env.BRANCH_NAME.startsWith('develop/')) {
                        env.PIPELINE_TYPE = 'develop'
                    } else if (env.BRANCH_NAME.startsWith('uat/')) {
                        env.PIPELINE_TYPE = 'uat'
                        env.DEPLOY_TAG = "${new Date().format('yyyyMMddHHmmss')}-uat-${env.GIT_COMMIT.substring(0, 7)}"
                    } else if (env.BRANCH_NAME == 'main') {
                        env.PIPELINE_TYPE = 'main'
                        env.DEPLOY_TAG = "${new Date().format('yyyyMMddHHmmss')}-release"
                    } else {
                        error("Branch ${env.BRANCH_NAME} is not managed by this pipeline.")
                    }
                }
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Package') {
            when {
                expression { env.PIPELINE_TYPE != 'develop' }
            }
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Sonarqube') {
            steps {
                script {
                    withSonarQubeEnv('SONAR_LATEST') {
                        sh """
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=${env.BRANCH_NAME.replace('/', '_')} \
                        -Dsonar.projectName=${env.BRANCH_NAME} \
                        -Dsonar.java.binaries=target/classes
                        """
                    }
                }
            }
        }

        stage('Push to Nexus') {
            when {
                expression { env.PIPELINE_TYPE in ['uat', 'main'] }
            }
            steps {
                script {
                    def artifactPath = "target/${NEXUS_ARTIFACT_ID}-0.0.1.jar"
                    def artifactPathVersion = "${ARTIFACT_VERS}-${env.DEPLOY_TAG}"
                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: NEXUS_URL,
                        groupId: NEXUS_GROUP,
                        version: artifactPathVersion,
                        repository: NEXUS_REPOSITORY,
                        credentialsId: NEXUS_CREDENTIALS_ID,
                        artifacts: [
                            [
                                artifactId: NEXUS_ARTIFACT_ID,
                                classifier: '',
                                file: artifactPath,
                                type: 'jar'
                            ]
                        ]
                    )
                }
            }
        }

        stage('Deploy and Tag') {
            when {
                expression { env.PIPELINE_TYPE in ['uat', 'main'] }
            }
            agent { label 'JDK8' }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'for-nexus', passwordVariable: 'pass', usernameVariable: 'user')]) {
                        sh """
                        curl -u $user:$pass -O \
                        http://${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/${NEXUS_GROUP}/${NEXUS_ARTIFACT_ID}/${ARTIFACT_VERS}/${NEXUS_ARTIFACT_ID}-${ARTIFACT_VERS}.jar
                        """
                        sh "java -jar ${NEXUS_ARTIFACT_ID}-${ARTIFACT_VERS}.jar &"
                    }
                    withCredentials([usernamePassword(credentialsId: 'for-github', usernameVariable: 'user', passwordVariable: 'pass')]) {
                        sh """
                        git config --global user.email "duy.nguyentadinh@gmail.com"
                        git config --global user.name "duydinhhh"
                        git tag -a ${env.DEPLOY_TAG} -m "Deployed ${env.PIPELINE_TYPE} environment"
                        git push https://$user:$pass@github.com/DuyDinhhh/BoardgameListingWebApp.git ${env.DEPLOY_TAG}
                        """
                    }
                }
            }
        }

        stage('Health Check') {
            when {
                expression { env.PIPELINE_TYPE in ['uat', 'main'] }
            }
            agent { label 'JDK8' }
            steps {
                script {
                    try {
                        sleep(20) // Wait for the application to start completely
                        def status = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://192.168.64.35:8080/actuator/health", returnStdout: true).trim()
                        if (status == '200') {
                            echo "Health Check Passed: HTTP Status ${status}"
                        } else {
                            echo "Health Check Failed: HTTP Status ${status}"
                            error("Health Check Failed")
                        }
                    } catch (e) {
                        echo "Health Check Exception: ${e.getMessage()}"
                        error("Health Check Failed")
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                if (env.PIPELINE_TYPE != 'develop') {
                    junit testResults: '**/surefire-reports/*.xml'
                    archiveArtifacts artifacts: 'target/*.jar'
                } else {
                    echo "Skipping artifact archiving for develop branch."
                }
            }
        }
        failure {
            echo 'Pipeline failed. Please review the logs.'
        }
    }
}
