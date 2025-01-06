pipeline {
    agent {label 'JDK17'}
    
     tools {
        jdk 'jdk17'
        maven "maven3"
    }
    environment{
        SCANNER_HOME = tool 'sonar-scanner'
        NEXUS_URL = "192.168.64.37:8081"
        NEXUS_CREDENTIALS_ID ="for-nexus"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_GROUP= "com/javaproject"
        NEXUS_ARTIFACT_ID="spring-petclinic"
        ARTIFACT_VERS="0.0.1"
    stages {
    
 		stage('Branch Validation') {
            steps {
                script {
                    if (env.BRANCH_NAME.startsWith("develop/")) {
                        env.PIPELINE_TYPE = "develop"
                    } else if (env.BRANCH_NAME.startsWith("uat/")) {
                        env.PIPELINE_TYPE = "uat"
                        env.DEPLOY_TAG = "${new Date().format('yyMMdd')}-uat-${env.GIT_COMMIT.substring(0, 7)}"
                    } else if (env.BRANCH_NAME == "main") {
                        env.PIPELINE_TYPE = "main"
                        env.DEPLOY_TAG = "${new Date().format('yyMMdd')}-release"
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
        stage('Push to nexus') {
          	when {
                expression { env.PIPELINE_TYPE in ['uat', 'main'] }
            }
            steps {
                script {
                    // Define artifact paths
                    def artifactPath = "target/${NEXUS_ARTIFACT_ID}-3.4.0-SNAPSHOT.jar"
                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: NEXUS_URL,
                        groupId: NEXUS_GROUP,
                        version: DEPLOY_TAG,
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
            	script{
            		 withCredentials([usernamePassword(credentialsId: 'for-nexus', passwordVariable: 'pass', usernameVariable: 'user')]) {
                    sh """
                        curl -u $user:$pass -O \
                        http://${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/${NEXUS_GROUP}/${NEXUS_ARTIFACT_ID}/${ARTIFACT_VERS}/${NEXUS_ARTIFACT_ID}-${ARTIFACT_VERS}.jar
                    """

                    sh "java -jar ${NEXUS_ARTIFACT_ID}-${ARTIFACT_VERS}.jar &"
                   }
            		
            		 sh """
                    git config --global user.email "duy.nguyentadinh@gmail.com"
                    git config --global user.name "duydinhhh"
                    git tag -a ${env.DEPLOY_TAG} -m "Deployed ${env.PIPELINE_TYPE} environment"
                    git push origin ${env.DEPLOY_TAG}
                    """
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
                            echo "Health Check Failedd: HTTP Status ${status}"
                            error("Health Check Failedd")
                        }
                    } catch (error) {
                        echo "Health Check Failed: ${error.getMessage()}"
                        error("Health Check Failed")
                    }
                }
            }
        }
    }

    
    post{
        success{
            junit testResults: '**/surefire-reports/*.xml'
            archiveArtifacts artifacts: 'target/*.jar'
        }
    }
}
pipeline {
    agent { label 'JDK17' }

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        NEXUS_URL = "192.168.64.37:8081"
        NEXUS_CREDENTIALS_ID = "for-nexus"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_GROUP = "com/javaproject"
        NEXUS_ARTIFACT_ID = "spring-petclinic"
        ARTIFACT_VERS = "0.0.1"
        DEPLOY_TAG = "${new Date().format('yyMMdd')}-uat-${env.GIT_COMMIT.substring(0, 7)}"
    }

    stages {
        stage('Git Checkout') {
            steps {
                echo "Cloning repository..."
                git url: 'https://github.com/DuyDinhhh/BoardgameListingWebApp.git', branch: 'main'
            }
        }

        stage('Compile') {
            steps {
                echo "Compiling the project..."
                sh 'mvn compile'
            }
        }

        stage('Unit Test') {
            steps {
                echo "Running unit tests..."
                sh 'mvn test'
            }
        }

        stage('SonarQube Scan') {
            steps {
                echo "Scanning with SonarQube..."
                withSonarQubeEnv('SONAR_LATEST') {
                    sh """
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=BoardgameListingWebApp \
                    -Dsonar.projectKey=BoardgameListingWebApp \
                    -Dsonar.java.binaries=target/classes
                    """
                }
            }
        }

        stage('Package') {
            steps {
                echo "Packaging the application..."
                sh 'mvn clean package'
            }
        }

        stage('Push to Nexus') {
            steps {
                script {
                    echo "Pushing artifact to Nexus repository..."
                    def artifactPath = "target/${NEXUS_ARTIFACT_ID}-${ARTIFACT_VERS}.jar"
                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: NEXUS_URL,
                        groupId: NEXUS_GROUP,
                        version: ARTIFACT_VERS,
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

        stage('Pull from Nexus and Deploy on Another Node') {
            when {
                expression { env.PIPELINE_TYPE in ['uat', 'main'] }
            }
            agent { label 'JDK8' }
            steps {
                script {
                    echo "Pulling artifact from Nexus and deploying on target node..."
                    withCredentials([usernamePassword(credentialsId: 'for-nexus', passwordVariable: 'pass', usernameVariable: 'user')]) {
                        sh """
                        curl -u $user:$pass -O \
                        http://${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/${NEXUS_GROUP}/${NEXUS_ARTIFACT_ID}/${ARTIFACT_VERS}/${NEXUS_ARTIFACT_ID}-${ARTIFACT_VERS}.jar
                        """
                        sh "java -jar ${NEXUS_ARTIFACT_ID}-${ARTIFACT_VERS}.jar &"
                    }

                    echo "Tagging deployment..."
                    sh """
                    git config --global user.email "duy.nguyentadinh@gmail.com"
                    git config --global user.name "duydinhhh"
                    git tag -a ${env.DEPLOY_TAG} -m "Deployed ${env.PIPELINE_TYPE} environment"
                    git push origin ${env.DEPLOY_TAG}
                    """
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
                    echo "Performing health check..."
                    try {
                        sleep(20) // Wait for the application to start completely
                        def status = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://192.168.64.35:8080/actuator/health", returnStdout: true).trim()
                        if (status == '200') {
                            echo "Health Check Passed: HTTP Status ${status}"
                        } else {
                            error("Health Check Failed: HTTP Status ${status}")
                        }
                    } catch (error) {
                        error("Health Check Failed: ${error.getMessage()}")
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully. Archiving results..."
            junit testResults: '**/surefire-reports/*.xml'
            archiveArtifacts artifacts: 'target/*.jar'
        }
        failure {
            echo "Pipeline failed. Please check the logs."
        }
    }
}

