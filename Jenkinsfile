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
        GIT_TOKEN = credentials('GIT_TOKEN')
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
        stage('Deploy in EC2') {
    when {
        branch pattern: "feature/.*", comparator: "REGEXP"
    }
    steps {
        script {
            sshagent(['ssh']) {
                sh """
                ssh -o StrictHostKeyChecking=no ubuntu@15.206.125.203 '
                    if sudo docker ps -a | grep -q "solar-system"; then
                        echo "Container found. Stopping..."
                        sudo docker stop solar-system && sudo docker rm solar-system
                        echo "Container stopped and removed."
                    fi

                    echo "Running new container..."
                    sudo docker run --name solar-system \\
                        -e MONGO_URI=${env.MONGO_URI} \\
                        -e MONGO_USERNAME=${env.MONGO_USERNAME} \\
                        -e MONGO_PASSWORD=${env.MONGO_PASSWORD} \\
                        -p 3000:3000 -d chakribaggam123/demo:${GIT_COMMIT}
                '
                """
            }
        }
    }
}
stage('integration testing') {
  when {
    branch comparator: 'REGEXP', pattern: 'feature/.*'
  }
  steps {
    withAWS(credentials: 'iam-login', region: 'ap-south-1') {
      sh '''
        echo "Running integration test script..."
        chmod +x integration-testing-ec2.sh
        ./integration-testing-ec2.sh
      '''
    }
  }
}
stage('updating the image'){
            when {
              branch 'PR*'
            }
            steps{
                sh 'git clone -b main https://github.com/porsche-org/gi.git'
        dir('gi/kubernetes') {
            sh '''
                ##### Replace Docker Tag #####
                git checkout main
                git checkout -b feature-$BUILD_ID
                sed -i "s#chakribaggam123.*#chakribaggam123/demo:$GIT_COMMIT#g" deployment.yml
                cat deployment.yml


                ##### Commit and Push to Feature Branch #####
                git config --global user.email "chakrachandb@gmail.com"
                git config --global user.name "chakribaggam456"
                git remote set-url origin https://$GIT_TOKEN@github.com/porsche-org/gi.git
                git add .
                git commit -am "Updated docker image"
                git push -u origin feature-$BUILD_ID
            '''
            }
            }
        }
        stage('raising pr for main'){
            when {
              branch 'PR*'
            }
            steps {
    sh """
    curl -s -o response.json -w "%{http_code}" -X POST \\
      https://api.github.com/repos/porsche-org/gi/pulls \\
      -H "Accept: application/vnd.github+json" \\
      -H "Authorization: token ${GIT_TOKEN}" \\
      -H "Content-Type: application/json" \\
      -d '{
        "title": "Updated Docker Image",
        "body": "Updated docker image in deployment manifest",
        "head": "feature-${BUILD_ID}",
        "base": "main",
        "assignees": ["chakribaggam456"]
      }'
    """
   }
}
stage("app deployed") {
    steps {
        timeout(time: 1, unit: 'DAYS') {
            input message: 'Is the new version of the app synced and deployed?', ok: 'Yes'
        }
    }
}

stage('DAST - OWASP ZAP') {
    when {
        branch 'PR*'
    }
    steps {
      catchError(buildResult: 'SUCCESS', message: 'no worries', stageResult: 'UNSTABLE'){
        sh '''
        ##### REPLACE below with Kubernetes http://IP_Address:30000/api-docs/ #####
        chmod 777 $(pwd)
        docker run -v $(pwd):/zap/wrk/:rw ghcr.io/zaproxy/zaproxy zap-api-scan.py \
        -t http://3.110.197.100:30000/api-docs/ \
        -f openapi \
        -r zap_report.html \
        -w zap_report.md \
        -J zap_json_report.json \
        -x zap_xml_report.xml
        -c zap_ignore_rules
        '''
       }
    }
}
stage('Upload - AWS S3') {
    when {
        branch 'PR*'
    }
    steps {
        withAWS(credentials: 'iam-login', region: 'ap-south-1') {
            sh '''
                ls -ltr
                mkdir reports-$BUILD_ID
                cp -rf coverage/ reports-$BUILD_ID/
                cp dependency* test-results.xml trivy*.* reports-$BUILD_ID/
                ls -ltr reports-$BUILD_ID/
            '''
            s3Upload(
                file: "reports-$BUILD_ID",
                bucket: 'porsche-ferrari',
                path: "jenkins-$BUILD_ID/"
            )
        }
    }
}
stage('Deploy to Prod?') {
    when {
        branch 'main'
    }
    steps {
        timeout(time: 1, unit: 'DAYS') {
            input message: 'Deploy to Production?', ok: 'YES! Let us try this on Production', submitter: 'admin'
        }
    }
}


    }

    post {
        always {
            script {
                if (fileExists('gi')) {
                    sh 'rm -rf gi'
                }
            }
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

