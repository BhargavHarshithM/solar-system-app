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
                git branch: 'feature/enabling-cicd', url: 'https://github.com/BhargavHarshithM/solar-system-app.git'
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

        stage('Unit Tests') {
            steps {
                sh 'echo Running Unit Tests...'
                sh 'npm test'
            }
        } 

        stage('Code Coverage') {
            steps {
                sh 'echo Running Code Coverage...'
                sh 'npm run coverage'
            }
        }   
    }
}

POST {
    always {
        junit allowEmptyResults: true, keepProperties: true, stdioRetention: '', testResults: 'dependency-check-junit.xml'
        junit allowEmptyResults: true, keepProperties: true, stdioRetention: '', testResults: 'test-results.xml'


        publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'dependency-check-jenkins.html', reportName: 'Dependency Check HTML Report', reportTitles: '', useWrapperFileDirectly: true])
        publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: 'coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage HTML Report', reportTitles: '', useWrapperFileDirectly: true])
    }
}
