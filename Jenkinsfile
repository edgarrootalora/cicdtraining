pipeline {
    agent any

    stages {
        stage('Test Jenkins') {
            steps {
                echo 'Jenkins está ejecutando correctamente el pipeline'
            }
        }

        stage('Test Docker') {
            steps {
                bat 'docker --version'
            }
        }
    }
}