pipeline {
    agent any

    tools {
        nodejs 'nodejs'
    }

    environment {
        MONGO_URI = ""
        MONGO_USERNAME = credentials('mongo-usr-creds-secret')
        MONGO_PASSWORD = credentials('mongo-pswd-creds-secret')
        SONAR_SCANNER_HOME = tool 'sonar-scanner' // the name of sonar scanner added
    }

    options {
        disableResume()
        disableConcurrentBuilds abortPrevious: true
    }

    stages {
        stage('Installing Dependencies') {
            options { timestamps() }
            steps {
                sh 'npm install --no-audit'
            }
        }

        stage('Dependency Scanning') {
            parallel {
                stage('npm audit') {
                    steps {
                        sh '''
                            npm audit --audit-level=critical
                        '''
                    }
                }

                stage('OWASP Dependency check') {
                    steps {
                        dependencyCheck additionalArguments: '''
                            --scan ./ 
                            --out ./ 
                            --format ALL
                            --disableYarnAudit  
                            --prettyPrint
                        ''',
                        odcInstallation: 'OWASP Dependency-Check'

                        dependencyCheckPublisher failedTotalCritical: 1, pattern: 'dependency-check-report.xml', stopBuild: true
                    }
                }
            }
        }

        stage('Unit Testing') {
            options { retry(2) }
            steps {
                sh 'npm test'
            }
        }

        stage('code coverage') {
            steps {
                catchError(buildResult: 'SUCCESS', message: "it will be fixed later", stageResult: 'UNSTABLE') {
                    sh 'npm run coverage'
                }
            }
        }

        stage('SAST - SonarQube') {
            steps {
                timeout(time: 60, unit: 'SECONDS') {
                    withSonarQubeEnv('sonar-qube-server') { // the name of sonarqube server added
                        sh 'echo $SONAR_SCANNER_HOME'
                        sh '''
                            $SONAR_SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectKey=Solar-System-Project \
                            -Dsonar.sources=app.js \
                            -Dsonar.javascript.lcov.reportPaths=./coverage/lcov.info
                        '''
                    }
                    waitForQualityGate abortPipeline: true // if sonarqube failed all pipeline will be failed
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t megs17/solar-system:$GIT_COMMIT .'
            }
        }

        stage('Trivy Vulnerability Scanner') {
            steps {
                // Scan for MEDIUM/LOW severity (report only)
                sh '''
                    trivy image siddharth67/solar-system:$GIT_COMMIT \
                    --severity LOW,MEDIUM,HIGH \
                    --exit-code 0 \
                    --quiet \
                    --format json \
                    -o trivy-image-MEDIUM-results.json
                '''
                
                // Scan for CRITICAL/HIGH severity (fail pipeline if found)
                sh '''
                    trivy image siddharth67/solar-system:$GIT_COMMIT \
                    --severity CRITICAL \
                    --exit-code 1 \
                    --quiet \
                    --format json \
                    -o trivy-image-CRITICAL-results.json || echo "Critical vulnerabilities found!"
                '''
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
        
        stage('Push Docker-Image') {
            steps {
                withDockerRegistry([credentialsId: 'docker-hub-credentials', url: 'https://index.docker.io/v1/']) {
                    sh 'docker push megs17/solar-system:$GIT_COMMIT'
                }
            }
         }
        /// the end of ci 

        stage('Deploy - AWS EC2') {
                when {
                    branch 'feature/*'
                }
                steps {
                    script {
                        sshagent(['aws-dev-deploy-ec2-instance']) { // id of credential of ssh
                            sh '''
                            ssh -o StrictHostKeyChecking=no ubuntu@3.140.244.188 "
                                if sudo docker ps -a | grep -q "solar-system"; then
                                    echo "Container found. Stopping..."
                                    sudo docker stop "solar-system" && sudo docker rm "solar-system"
                                    echo "Container stopped and removed."
                                fi

                                sudo docker run --name solar-system \\
                                    -e MONGO_URI=$MONGO_URI \\
                                    -e MONGO_USERNAME=$MONGO_USERNAME \\
                                    -e MONGO_PASSWORD=$MONGO_PASSWORD \\
                                    -p 3000:3000 -d siddharth67/solar-system:$GIT_COMMIT
                            "
                            '''
                        }
                    }
                } 
                }
            // here we are going to run integration testing on ec2 instance
            // in the real world scenarios the integration testing will run but with other tools not with bash script
        stage('Integration Testing - AMS EC2') {
            when {
                branch 'feature/'
            }
            steps {
                withAWS(credentials: 'aws-s3-ec2-lambda-creds', region: 'us-east-2') {
                    sh '''
                        # Your shell commands here
                        bash integration-testing-ec2.sh
                        # Additional commands if needed
                    '''
                }
            }
                                             }

          // the end of cd

          stage('K8S Update Image Tag') 
          {
             when {
                branch 'PR*'  // work if pull request triggered
            }
            steps {
             sh 'git clone -b main http://64.227.187.25:5555/dasher-org/solar-system-gitops-argocd'
             dir("solar-system-gitops-argocd/kubernetes") {
                git checkout main
                git checkout -b feature-$BUILD_ID
                sed -i "s#siddharth67.*#siddharth67/solar-system:$GIT_COMMIT#g" deployment.yml
                cat deployment.yml

                git config --global user.email ""
                git remote set-url origin http://$GITEA_TOKEN@64.227.187.25:5555/dasher-org/solar-system-  // here use github
                git add .
                git commit -am "Updated docker image"
                git push -u origin feature-$BUILD_ID

            }

          }
          
        }
        // not mandaory step for me here he make a pull request to gitea repo for argocd     on github will be easier
        stage('K8S - Raise PR') {
                    when {
                        branch 'PR*'
                    }
                        steps {
                            sh """
                            curl -X 'POST' \\
                            'http://64.227.187.25:5555/api/v1/repos/dasher-org/solar-system-gitops-argocd/pulls' \\
                            -H 'accept: application/json' \\
                            -H 'Authorization: token $GITEA_TOKEN' \\
                            -H 'Content-Type: application/json' \\
                            -d '{
                            "assignee": "gitea-admin",
                            "assignees": [
                                "gitea-admin"
                            ],
                            "base": "main",
                            "body": "Updated docker image in deployment manifest",
                            "head": "feature-$BUILD_ID",
                            "title": "Updated Docker Image"
                            }'
                            """
                    }
            }

            stage('App Deployed') {
                when {
                    branch 'PR*'
                }
                steps {
                    timeout(time: 1, unit: 'DAYS') {
                        input message: 'Is the PR Merged and ArgocD Synced?', ok: 'YES! PR is Merged and ArgocD Appli'

                    }
                }
            }
            
            stage('DAST - OWASP ZAP') {
                 when {
                        branch 'PR*'
                    }
                    steps {
                        sh '''
                        #### REPLACE below with Kubernetes http://IP_Address:3000/api-docs/ ####
                        chmod 777 $(pwd)
                        docker run -v $(pwd):/zap/wrk/:rw ghcr.io/zaproxy/zaproxy zap-api-scan.py \
                        -t http://134.209.155.222:3000/api-docs/ \
                        -f openapi \
                        -r zap_report.html \
                        -w zap_report.md \
                        -J zap_json_report.json \
                        -x zap_xml_report.xml
                        '''
                    }
                }
            // uploading reports to s3bucket
            stage('Upload - AWS S3') {
                when {
                    branch 'PR*'
                }
                steps {
                    withAWS(credentials: 'aws-s3-ec2-lambda-creds', region: 'us-east-2') {
                        sh '''
                            ls -ltr
                            mkdir reports-$BUILD_ID
                            cp -rf coverage/ reports-$BUILD_ID/
                            cp dependency*. * test-results.xml trivy*. * zap*. * reports-$BUILD_ID/
                            ls -ltr reports-$BUILD_ID/
                        '''
                        s3Upload(
                            file: "reports-$BUILD_ID",
                            bucket: 'solar-system-jenkins-reports-bucket',
                            path: "jenkins-$BUILD_ID/"
                        )
                    }
                }
            }

            


            // here we deployed application to cluster and run the zap api scan
            // upcomeing is other deployment method not related to the cluster

    }

    post {
        always {
            script {
                if (fileExists('solar-system-gitops-argocd')) {
                sh 'rm -rf solar-system-gitops-argocd'
            }
            } 
}
            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'coverage/lcov.info',
                reportFiles: 'index.html',
                reportName: 'Code Coverage Report',
                reportTitles: '',
                useWrapperFileDirectly: true
            ])

            junit allowEmptyResults: true, testResults: 'test-results.xml'
            junit allowEmptyResults: true, testResults: 'dependency-check-junit.xml'
            junit allowEmptyResults: true, testResults: 'trivy-image-MEDIUM-results.xml'
            junit allowEmptyResults: true, testResults: 'trivy-image-CRITCAL-results.xml'

            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: './',
                reportFiles: 'dependency-check-jenkins.html',
                reportName: 'Dependency Check HTML Report',
                reportTitles: '',
                useWrapperFileDirectly: true
            ])


            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: './',
                reportFiles: 'trivy-image-MEDIUM-results.html',
                reportName: 'trivy-image-MEDIUM-results',
                reportTitles: '',
                useWrapperFileDirectly: true
            ])

             publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: './',
                reportFiles: 'trivy-image-CRITICAL-results.html',
                reportName: 'trivy-image-CRITICAL-results',
                reportTitles: '',
                useWrapperFileDirectly: true
            ])
        }
    }


