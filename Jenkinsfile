pipeline {

    agent any
    tools {
        nodejs 'node'
    }

    environment {
        MONGO_URI = 'mongodb+srv://supercluster.d83jj.mongodb.net/superData'
        MONGO_DB_CREDS = credentials('mongo-db-creds')
        MONGO_USERNAME = credentials('mongo-db-username')
        MONGO_PASSWORD = credentials('mongo-db-password')
        SONAR_SCANNER_HOME = tool 'sonarqube-scanner-610'
        GITHUB_TOKEN = credentials('git-pat-token')
    }

    stages {
        stage('Install Dependencies') {
            steps {
                script {
                    sh 'npm install --no-audit'
                }
            }
        }

        stage('Dependency Scanning') {
            parallel {
                stage('NPM Dependency Audit') {
                    steps {
                        script {
                            sh '''
                               npm audit --audit-level=critical || true
                               echo $?
                            '''
                        }
                    }
                }
                stage('OWASP Dependency Check') {
                    steps {
                        script {
                            dependencyCheck additionalArguments: '''
                                --scan \'./\' 
                                --out \'./\'  
                                --format \'ALL\' 
                                --disableYarnAudit \
                                --prettyPrint''', odcInstallation: 'OWASP-DepCheck-11'

                            dependencyCheckPublisher failedTotalCritical: 3, pattern: 'dependency-check-report.xml', stopBuild: true
                        }
                    }
                }
            }
        }

        stage('Unit Tests') {
            steps {
                script {
                    sh 'echo Colon-Separated - $MONGO_DB_CREDS'
                    sh 'echo Username - $MONGO_DB_CREDS_USR'
                    sh 'echo Password - $MONGO_DB_CREDS_PSW'
                    sh 'npm test'
                }
            }
        }

        stage('Code Coverage') {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', message: 'Oops! it will be fixed in future releases', stageResult: 'UNSTABLE') {
                        sh 'npm run coverage'
                    }
                }
            }
        }

        stage('SAST - SonarQube') {
            steps {
                script {
                    timeout(time: 60, unit: 'SECONDS') {
                        withSonarQubeEnv('sonar-qube-server') {
                            sh '''
                                $SONAR_SCANNER_HOME/bin/sonar-scanner \
                                  -Dsonar.projectKey=orbit-engine \
                                  -Dsonar.projectName=orbit-engine \
                                  -Dsonar.sources=app.js \
                                  -Dsonar.javascript.lcov.reportPaths=./coverage/lcov.info
                            '''
                        }
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t chinmayapradhan/orbit-engine:$GIT_COMMIT ."
                }
            }
        }

        stage('Trivy Vulnerability Scan') {
            steps {
                script {
                    sh """
                        trivy image chinmayapradhan/orbit-engine:$GIT_COMMIT \
                          --severity LOW,MEDIUM,HIGH \
                          --exit-code 0 \
                          --quiet \
                          --format json -o trivy-image-MEDIUM-results.json

                        trivy image chinmayapradhan/orbit-engine:$GIT_COMMIT \
                          --severity CRITICAL \
                          --exit-code 0 \
                          --quiet \
                          --format json -o trivy-image-CRITICAL-results.json
                    """
                }
            }
            post {
                always {
                    sh '''
                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/html.tpl" \
                            --output trivy-image-MEDIUM-results.html trivy-image-MEDIUM-results.json

                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/html.tpl" \
                            --output trivy-image-CRITICAL-results.html trivy-image-CRITICAL-results.json

                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/junit.tpl" \
                            --output trivy-image-MEDIUM-results.xml  trivy-image-MEDIUM-results.json 

                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/junit.tpl" \
                            --output trivy-image-CRITICAL-results.xml trivy-image-CRITICAL-results.json
                    '''
                }
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-creds', url: '') {
                        sh "docker push chinmayapradhan/orbit-engine:$GIT_COMMIT"
                    }
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    sshagent(['ec2-server-key']) {
                        sh '''
                            ssh -o StrictHostKeyChecking=no ec2-user@18.217.135.213 "
                                if sudo docker ps -a | grep -q "solar-system"; then
                                    echo "Container found. Stopping..."
                                    sudo docker stop "solar-system" && sudo docker rm "solar-system"
                                    echo "Container stopped and removed."
                                fi
                                    sudo docker run --name solar-system \
                                        -e MONGO_URI=$MONGO_URI \
                                        -e MONGO_USERNAME=$MONGO_USERNAME \
                                        -e MONGO_PASSWORD=$MONGO_PASSWORD \
                                        -p 3000:3000 -d chinmayapradhan/orbit-engine:$GIT_COMMIT
                            "
                        '''
                    }
                }
            }
        }

        stage('Integration Testing - AWS EC2') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: 'us-east-2') {
                        sh 'bash integration-testing-ec2.sh'
                    }
                }
            }
        }

        stage('K8S - Update Image Tag') {
            steps {
                script {
                    sh 'git clone -b main https://github.com/chinmaya10000/kubernetes-manifest.git'
                    dir("kubernetes-manifest/solar-system") {
                        sh '''
                           #### Replace Docker Tag ####
                           git checkout main
                           git checkout -b feature-$BUILD_ID
                           sed -i "s#chinmayapradhan.*#chinmayapradhan/orbit-engine:$GIT_COMMIT#g" deployment.yaml

                           #### Commit and Push to Feature Branch ####
                           git config --global user.name "jenkins"
                           git config --global user.email "jenkins@dasher.com"
                           git remote set-url origin https://${GITHUB_TOKEN}@github.com/chinmaya10000/kubernetes-manifest.git
                           git add .
                           git commit -am "Updated docker image"
                           git push -u origin feature-$BUILD_ID
                        '''
                    }
                }
            }
        }

        stage('k8s - Raise PR') {
            steps {
                script {
                    sh """
                       curl -X POST -H "Authorization: token ${GITHUB_TOKEN}" \
                       -H "Accept: application/vnd.github.v3+json" \
                       'https://api.github.com/repos/chinmaya10000/kubernetes-manifest/pulls' \
                       -d '{
                            "assignee": "git-admin",
                                "assignees": [
                                    "git-admin"
                                ],
                            "base": "main",
                            "body": "Updated docker image in deployment manifest",
                            "head": "feature-$BUILD_ID",
                            "title": "Updated Docker Image"
                        }'
                    """
                }
            }
        }

        stage('App Deployed?') {
            steps {
                script {
                    timeout(time: 1, unit: 'DAYS') {
                        input message: 'Is the PR Merged and ArgoCD Synced?', ok: 'YES! PR is Merged and ArgoCD Application is Synced'
                    }
                }
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                script {
                    sh '''
                        #### REPLACE below with Kubernetes http://IP_Address:30000/api-docs/ #####
                        chmod 777 $(pwd)
                        docker run -v $(pwd):/zap/wrk/:rw ghcr.io/zaproxy/zaproxy zap-api-scan.py \
                        -t http://a6e275847d50f4ef3b1758b2be5639f2-1051821613.us-east-2.elb.amazonaws.com:3000/api-docs/ \
                        -f openapi \
                        -r zap_report.html \
                        -w zap_report.md \
                        -J zap_json_report.json \
                        -x zap_xml_report.xml 
                    '''
                }
            }
        }

        stage('Upload - AWS S3') {
            steps {
                withAWS(credentials: 'aws-creds', region: 'us-east-2') {
                    sh  '''
                        ls -ltr
                        mkdir reports-$BUILD_ID
                        cp -rf coverage/ reports-$BUILD_ID/
                        cp dependency*.* test-results.xml trivy*.* zap*.* reports-$BUILD_ID/
                        ls -ltr reports-$BUILD_ID/
                    '''
                    s3Upload(
                        file:"reports-$BUILD_ID", 
                        bucket:'orbit-engine-jenkins-reports', 
                        path:"jenkins-$BUILD_ID/"
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
                if (fileExists('kubernetes-manifest')) {
                    sh 'rm -rf kubernetes-manifest'
                }
            }

            junit allowEmptyResults: true, stdioRetention: '', testResults: 'dependency-check-junit.xml'
            junit allowEmptyResults: true, stdioRetention: '', testResults: 'test-results.xml'
            junit allowEmptyResults: true, stdioRetention: '', testResults: 'trivy-image-CRITICAL-results.xml'
            junit allowEmptyResults: true, stdioRetention: '', testResults: 'trivy-image-MEDIUM-results.xml'

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './', reportFiles: 'dependency-check-jenkins.html', reportName: 'Dependency Check HTML Report', reportTitles: '', useWrapperFileDirectly: true])

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage HTML Report', reportTitles: '', useWrapperFileDirectly: true])
            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './', reportFiles: 'trivy-image-CRITICAL-results.html', reportName: 'Trivy Image Critical Vul Report', reportTitles: '', useWrapperFileDirectly: true])

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './', reportFiles: 'trivy-image-MEDIUM-results.html', reportName: 'Trivy Image Medium Vul Report', reportTitles: '', useWrapperFileDirectly: true])

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './', reportFiles: 'zap_report.html', reportName: 'DAST - OWASP ZAP Report', reportTitles: '', useWrapperFileDirectly: true])
        }
    }
}
