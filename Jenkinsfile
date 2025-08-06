pipeline {
    agent any

    tools {
        nodejs 'nodejs-22.6.0'
    }

    environment {
        MONGO_URI = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
        MONGO_USERNAME = credentials('mongo-username')
        MONGO_PASSWORD = credentials('mongo-password')
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
                            npm audit --audit-level=critical || true
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

                        dependencyCheckPublisher failedTotalCritical: 5, pattern: 'dependency-check-report.xml', stopBuild: true

                    }
                }
            }
        }

        stage('unit testing') {
            steps {
                    sh 'npm test'
            }
        }
        stage('code coverage') {
            steps {
                catchError(buildResult: 'SUCCESS', message: 'no worries', stageResult: 'UNSTABLE') {
                    sh 'npm run coverage'
                }
                publishHTML([
                            allowMissing: true,
                            alwaysLinkToLastBuild: true,
                            icon: '',
                            keepAll: true,
                            reportDir: 'coverage/lcov-report',
                            reportFiles: 'index.html',
                            reportName: 'code coverage Report',
                            reportTitles: '',
                            useWrapperFileDirectly: true
                        ])
            }
        }
    }
    post {
        always {
            junit allowEmptyResults: true, testResults: 'dependency-check-junit.xml'
            junit allowEmptyResults: true, testResults: 'test-results.xml'
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

