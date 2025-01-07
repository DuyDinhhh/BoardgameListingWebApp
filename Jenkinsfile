@Library('share_library_build') _
pipeline {
    agent any
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
