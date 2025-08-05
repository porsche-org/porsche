pipeline{
    agent any
    tools{
        nodejs 'nodejs-22.6.0'
    }
    stages{
        stage('installing dependencies'){
            steps{
                sh 'npm install --no-audit'
            }
        }
    }
}
