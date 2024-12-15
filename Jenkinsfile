pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                bat 'echo Building the project...'
                bat 'powershell Compress-Archive -Path * -DestinationPath website.zip'
            }
        }
    }
}
