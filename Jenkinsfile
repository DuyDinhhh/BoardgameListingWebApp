@Library('share_library_build') _
pipeline {
    agent {label 'JDK17'}
    stages {
        stage('Test Shared Library') {
            steps {
                script {
                    buildSpringboot()
                }
            }
        }
    }
}
