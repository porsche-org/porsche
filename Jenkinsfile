pipeline {
    agent any

    tools {
        nodejs 'nodejs-22.6.0'
    }

    environment {
        MONGO_URI = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
    }

    stages {
        stage('installing dependencies') {
            steps {
                sh 'npm install --no-audit'
            }
        }

        stage('dependency scanning') {
            parallel {
                stage('npm audit scan') {
                    steps {
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

                        junit allowEmptyResults: true, testResults: 'dependency-check-junit.xml'

                        publishHTML([
                            allowMissing: true,
                            alwaysLinkToLastBuild: true,
                            icon: '',
                            keepAll: true,
                            reportDir: './',
                            reportFiles: 'dependency-check-jenkins.html',
                            reportName: 'HTML Report',
                            reportTitles: '',
                            useWrapperFileDirectly: true
                        ])
                    }
                }
            }
        }

        stage('unit testin') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'envcred', passwordVariable: 'MONGO_PASSWORD', usernameVariable: 'MONGO_USERNAME')]) {
                    sh 'npm test'
                }
                junit allowEmptyResults: true, testResults: 'test-results.xml'
            }
        }
    }
}
