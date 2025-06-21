pipeline {
    agent any
    tools { nodejs 'NodeJS-24' }

    environment {
      MONGO_URI = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
    }

    options {
      disableConcurrentBuilds abortPrevious: true
      disableResume()
    }
    
    stages {
        stage('Check Node version') {
            steps {
                sh 'echo Printing Node Version'
                sh 'node -v'
                sh 'echo Printing NPM Version'
                sh 'npm -v'
            }
        }
        
        stage('Git Checkout'){
            steps {
                sh 'echo Cloning Git Repository'
                git branch: 'main', url: 'https://github.com/BhargavHarshithMudragiri/solar-system.git'
            }
        }
        
        stage("Install Node Dependencies") {
            options {
               timestamps()
            }
            steps {
                sh 'echo Installing Node Dependencies...'
                sh ''' npm install --no-audit
                       echo $?
                    '''
                }
            }
        
        stage('Run Dependency Checks'){
            parallel {
                stage('NPM Dependency Audit'){
                    steps {
                        sh 'npm audit --audit-level=critical'
                    }
                }
                
                stage('OWASP Dependency Check') {
                    steps {
                       dependencyCheck additionalArguments: '''
                       --scan \'./\'
                       --out \'./\'
                       --format \'ALL\'
                       --prettyPrint ''' , odcInstallation: 'DPCheck-12'
                       
                       dependencyCheckPublisher failedTotalCritical: 1, pattern: 'dependency-check-report.xml'
                    }
                }
            }
        }

        stage('Run Unit Tests') {
            options {
              retry(2)
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'mongo-db-creds', passwordVariable: 'MONGO_PASSWORD', usernameVariable: 'MONGO_USERNAME')]) {
                  sh 'npm test'
                }
            }
        }

        stage('Run Code Coverage') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'mongo-db-creds', passwordVariable: 'MONGO_PASSWORD', usernameVariable: 'MONGO_USERNAME')]) {
                  catchError(buildResult: 'SUCCESS', message: 'Oops! This will be fixed in coming release.', stageResult: 'UNSTABLE') {
                    sh 'npm run coverage'
                }
                }
            }
        }

        stage('SAST-Sonarqube'){
            steps {
                withCredentials([usernamePassword(credentialsId: 'sonarqube-creds', passwordVariable: 'SONAR_PASSWORD', usernameVariable: 'SONAR_USERNAME')]) {
                    sh '''
                    echo "Running SonarQube Analysis"
                    sonar-scanner \
                    -Dsonar.projectKey=solar-system \
                    -Dsonar.sources=. \
                    -Dsonar.host.url=http://localhost:9000 \
                    -Dsonar.login=$SONAR_USERNAME \
                    -Dsonar.password=$SONAR_PASSWORD
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'echo Building Docker Image'
                sh 'docker build -t bhargavharshith/solar-system:$GIT_COMMIT .'
            }
        }

        stage('Trivy Vulnerability Scan') {
            steps {
                sh 'echo Running Trivy Vulnerability Scan'
                sh '''
                   trivy image bhargavharshith/solar-system:$GIT_COMMIT \
                   --severity LOW,MEDIUM,HIGH \
                   --exit-code 0 \
                   --quiet \
                   --format json -o trivy-image-MEDIUM-results.json 

                   trivy image bhargavharshith/solar-system:$GIT_COMMIT \
                   --severity CRITICAL \
                   --exit-code 1 \
                   --quiet \
                   --format json -o trivy-image-CRITICAL-results.json
                   '''
            }

            POST {
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
                         --output trivy-image-MEDIUM-junit.xml trivy-image-MEDIUM-results.json

                         trivy convert \
                         --format template --template "@/usr/local/share/trivy/templates/junit.tpl" \
                         --output trivy-image-CRITICAL-junit.xml trivy-image-CRITICAL-results.json
                      '''
                }
            }
        }

        stage('Push Docker Image To Registry') {
            steps {
                withDockerRegistry(credentialsId: 'dockerhub-creds', url: 'https://index.docker.io/v1/') {
                    sh 'echo Pushing Docker Image to Docker Hub'
                    sh 'docker push bhargavharshith/solar-system:$GIT_COMMIT'
                }
            }
        }

        stage('Deploy - AWS EC2') {
            when {
                branch 'main'
            }
                steps {
                    script {
                    sshagent(['aws-ec2-s3-lamdba-creds']) {
                            sh '''
                            ssh -o StrictHostKeyChecking=no ubuntu@52.201.213.58"
                            if sudo docker ps -a | grep -q "solar-system"; then
                                echo 'Container exists, stopping and removing...'
                                sudo docker stop solar-system && sudo docker rm solar-system
                                echo 'Container stopped and removed.'
                            fi

                            sudo docker run -d --name solar-system \
                            -e MONGO_URI=$MONGO_URI \
                            -e MONGO_USERNAME=$MONGO_USERNAME \
                            -e MONGO_PASSWORD=$MONGO_PASSWORD \
                            -p 3000:3000 -d bhargavharshith/solar-system:$GIT_COMMIT "
                            '''
                    }
                }
            }
        }

        stage('Integration tests') {
            when { branch 'main' }
            steps {
                sh 'printenv | grep -i branch'
                withAWS(credentials: 'aws-ec2-s3-lamdba-creds', region: 'us-east-1') {
                    sh '''
                    echo "Running Integration Tests"
                    bash Integration-Testing-EC2.sh 
                    '''
                }
            }
        }
    }
}

POST {
    always {
        junit allowEmptyResults: true, keepProperties: true, stdioRetention: '', testResults: 'dependency-check-junit.xml'
        junit allowEmptyResults: true, keepProperties: true, stdioRetention: '', testResults: 'test-results.xml'
        junit allowEmptyResults: true, keepProperties: true, stdioRetention: '', testResults: 'trivy-image-MEDIUM-junit.xml'
        junit allowEmptyResults: true, keepProperties: true, stdioRetention: '', testResults: 'trivy-image-CRITICAL-junit.xml'

        publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'dependency-check-jenkins.html', reportName: 'Dependency Check HTML Report', reportTitles: '', useWrapperFileDirectly: true])

        publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: 'coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage HTML Report', reportTitles: '', useWrapperFileDirectly: true])

        publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'trivy-image-CRITICAL-results.html', reportName: 'Trivy Image CRITICAL Vul Report', reportTitles: '', useWrapperFileDirectly: true])
        
        publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'trivy-image-MEDIUM-results.html', reportName: 'Trivy Image MEDIUM Vul Report', reportTitles: '', useWrapperFileDirectly: true])
    }
}
