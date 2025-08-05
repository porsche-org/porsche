pipeline{
    agent any
    tools{
        nodejs 'nodejs-22.6.0'
    }
    stages{
        stage('demo'){
            steps{
                sh '''
                   node -v
                   npm -v
                '''
            }
        }
    }
}
