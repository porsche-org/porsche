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
        stage('dependency scanning'){
            parallel{
                stage('dependency scanniing'){
            steps{
                sh '''
                   npm audit --audit-level=critical
                   echo $?
                '''
            }
        }
        stage('OWASP Dependency Check') {
                steps {
                    dependencyCheck additionalArguments: '''
                        --scan './'
                        --out './'
                        --format 'ALL'
                        --prettyPrint
                    ''', odcInstallation: 'depcheck'
                    dependencyCheckPublisher failedTotalCritical: 1, pattern: 'dependency-check-report.xml', stopBuild: true
                }
            }
            }
        }
    }
}
