pipeline {
    agent any

    tools {
        nodejs 'nodejs-22.6.0'
    }

    environment {
        MONGO_URI = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
        MONGO_USERNAME = credentials('mongo-username')
        MONGO_PASSWORD = credentials('mongo-password')
        SONAR_SCANNER_HOME = tool 'sonarqube-scanner-610'
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
                            --disableYarnAudit \
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
                    reportName: 'Code Coverage Report',
                    reportTitles: '',
                    useWrapperFileDirectly: true
                ])
            }
        }

        stage('sonarqube analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SONAR_SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=porsche \
                        -Dsonar.sources=app.js \
                        -Dsonar.javascript.lcov.reportPaths=./coverage/lcov.info
                    '''
                }
                waitForQualityGate abortPipeline: true
            }
        }

        stage('docker build') {
            steps {
                sh 'docker build -t chakribaggam123/demo:$GIT_COMMIT .'
            }
        }

        stage('trivy scan') {
            steps {
                sh '''
                    echo "🔍 Scanning Docker image with Trivy: chakribaggam123/demo:$GIT_COMMIT"

                    # Scan for low/medium/high vulnerabilities
                    trivy image chakribaggam123/demo:$GIT_COMMIT \
                        --severity LOW,MEDIUM,HIGH \
                        --exit-code 0 \
                        --quiet \
                        --format json -o image-medium-results.json 

                    # Scan for critical vulnerabilities
                    trivy image chakribaggam123/demo:$GIT_COMMIT \
                        --severity CRITICAL \
                        --exit-code 0 \
                        --quiet \
                        --format json -o image-critical-results.json
                '''
            }
            post {
                always {
                    sh '''
                        echo "📄 Converting Trivy JSON reports to HTML and JUnit format"

                        # Convert to HTML reports
                        trivy convert --format template --template "@/usr/local/share/trivy/templates/html.tpl" \
                            --output image-medium-results.html image-medium-results.json

                        trivy convert --format template --template "@/usr/local/share/trivy/templates/html.tpl" \
                            --output image-critical-results.html image-critical-results.json

                        # Convert to JUnit XML reports
                        trivy convert --format template --template "@/usr/local/share/trivy/templates/junit.tpl" \
                            --output image-medium-results.xml image-medium-results.json

                        trivy convert --format template --template "@/usr/local/share/trivy/templates/junit.tpl" \
                            --output image-critical-results.xml image-critical-results.json
                    '''

                    junit allowEmptyResults: true, testResults: 'image-*.xml'
                }
            }
        }
        stage('docker push'){
            steps{
                withDockerRegistry(credentialsId: 'docker-cred', url: "") {
                    sh 'docker push chakribaggam123/demo:$GIT_COMMIT'
                }
            }
        }
    }

    post {
        always {
            // JUnit reports
            junit allowEmptyResults: true, testResults: 'dependency-check-junit.xml'
            junit allowEmptyResults: true, testResults: 'image-critical-results.xml'
            junit allowEmptyResults: true, testResults: 'image-medium-results.xml'
            junit allowEmptyResults: true, testResults: 'test-results.xml'

            // Trivy HTML reports
            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                icon: '',
                keepAll: true,
                reportDir: './',
                reportFiles: 'image-medium-results.html',
                reportName: 'HTML Report of medium trivy',
                reportTitles: '',
                useWrapperFileDirectly: true
            ])

            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                icon: '',
                keepAll: true,
                reportDir: './',
                reportFiles: 'image-critical-results.html',
                reportName: 'HTML Report of critical trivy',
                reportTitles: '',
                useWrapperFileDirectly: true
            ])

            // OWASP Dependency Check HTML report
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

