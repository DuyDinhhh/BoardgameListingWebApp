@Library(['share_library_build', 'share_library_test', 'share_library_deploy']) _

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
        ARTIFACT_VERS = "1.${env.BUILD_ID}"
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
                script {
                    buildSpringboot()
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    unitTestJava()
                }
            }
        }

        stage('Package') {
            when {
                expression { env.PIPELINE_TYPE != 'develop' }
            }
            steps {
                script {
                    packageSpringboot()
                }
            }
        }

        stage('Sonarqube') {
            steps {
                script {
                    qualitySonarCheck(env)
                }
            }
        }

        stage('Push to Nexus') {
            when {
                expression { env.PIPELINE_TYPE in ['uat', 'main'] }
            }
            steps {
                script {
                    pushArtifactNexusJava(env)
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
                    pullArtifactNexusJava(env)
                    deployJava(env)
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
                    healthCheck()
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
            mail to: "duy2004tv@gmail.com",
                 subject: "${JOB_NAME} - Build # ${BUILD_NUMBER} - SUCCESS!",
                 body: "Check console output at ${BUILD_URL} to view the results."
        }
        failure {
            echo 'Pipeline failed. Please review the logs.'
            mail to: "duy2004tv@gmail.com",
                 subject: "${JOB_NAME} - Build # ${BUILD_NUMBER} - FAILURE!",
                 body: "Check console output at ${BUILD_URL} to view the results."
        }
    }
}
