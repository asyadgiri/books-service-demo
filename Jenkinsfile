#!/usr/bin/env groovy

/* groovylint-disable-next-line CompileStatic */
@Library('piper-lib-os') _   //to use the newman docker image

pipeline {
    agent any
    options {
        buildDiscarder logRotator(daysToKeepStr: '1', numToKeepStr: '3')
        skipDefaultCheckout()
        timestamps()
    }
    stages {
        stage('Init') {
            steps {
                lock(resource: "${env.JOB_NAME}/5", inversePrecedence: true) {
                    milestone 5
                    deleteDir()
                    // sh 'git clone https://github.com/AshwinSYadgiri/books-service-demo.git ./ '
                    checkout scm
                }
            }
        }

        stage('Build ') {
            when {
                anyOf {
                    branch 'master'
                    branch 'PR-*'
                }
            }

            steps {
                lock(resource: "${env.JOB_NAME}/10", inversePrecedence: true) {
                    milestone 10
                    mtaBuild(script: this, buildTarget: 'CF')              // We can also use mbt build if MBT tool is installed in the Jenkins
                    stash name: 'deployment', includes: '*.mtar, mta.yaml'
                    sh 'ls -la'
                }

                    //Publish Test Results
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                    jacoco execPattern: '**/target/coverage-reports/*.exec'

                    //Sonar checks
                    sonarExecuteScan(script: this, sonarTokenCredentialsId: 'sonarId', serverUrl: 'https://sonar.tools.sap/')
                    //check for the quality gate
                    script { 
			            withSonarQubeEnv('sonar'){ 
			                echo 'Sonar Scanner...'
			            }
			            timeout(time: 10, unit: 'MINUTES') {
             			//waitForQualityGate(credentialsId: "sonarId",abortPipeline: false)
			            }
			        }
            }
        }

        stage('Code quality checks') {
            when {
                anyOf {
                    branch 'master'
                    branch 'PR-*'
                }
            }

            steps {
                lock(resource: "${env.JOB_NAME}/15", inversePrecedence: true) {
                    milestone 15
                    sh 'ls -la'

                    //code checks for JAVA - PMD
                    sh 'mvn  -pl !integration-tests -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn --batch-mode org.apache.maven.plugins:maven-pmd-plugin:3.14.0:check'

                    recordIssues failOnError: true, failedTotalHigh: 5, tool: pmdParser(pattern: '**/target/pmd.xml')

                    //code checks for JS - ESLINT
                    npmExecuteLint(script: this, install: true, runScript: 'lint', defaultNpmRegistry: 'https://registry.npmjs.org/', failOnError: false)

                }
            }
        }

        //Integration Stage
        stage('Integration Stage') {
            when {
                anyOf {
                    branch 'master'
                    branch 'PR-*'
                }
            }
            steps {
                lock(resource: "${env.JOB_NAME}/17", inversePrecedence: true) {
                    milestone 17
                    sh 'mvn clean install -s settings.xml'
                    junit allowEmptyResults: true, testResults:'**/integration-tests/target/surefire-reports/*.xml'
                    jacoco execPattern: '**/target/coverage-reports/ITjacoco.exec'
                }
            }
        }

        stage('Deployment- Dev Space') {
            when {
                branch 'master'
            }

            steps {
                lock(resource: "${env.JOB_NAME}/22", inversePrecedence: true) {
                    milestone 22
                    echo '....Dev Deployment ..'
                    cloudFoundryDeploy(script: this, apiEndpoint: 'https://api.cf.us10-001.hana.ondemand.com/', buildTool: 'mta', deployTool: 'mtaDeployPlugin',  space: 'dev', org: 'ac864addtrial', cfCredentialsId: 'cf_credential_id')

                    //healthcheck
                    script {
                        def checkUrl = 'https://demo-service-srv-dev.cfapps.us10-001.hana.ondemand.com/'
                        def statusCode = curl(checkUrl)
                        if (statusCode != '200') {
                            error "Health check failed: ${statusCode}"
                       } else {
                            echo "Health check for ${checkUrl} successful"
                        }
                    }
                }
            }
        }

        stage('API Tests') {
            when {
                branch 'master'
            }

            steps {
                lock(resource: "${env.JOB_NAME}/25", inversePrecedence: true) {
                    milestone 25
                    echo 'API Tests - Newman...'
                    //execute newman(postman) script
                    newmanExecute(script: this, failOnError: true,
                        newmanInstallCommand: 'npm install newman --global newman-reporter-htmlextra --quiet',
                        newmanCollection: 'scripts/demo-scripts.postman_collection.json',
                        newmanEnvironment: 'scripts/demoEnv.postman_environment.json',
                        runOptions: ['run', 'scripts/demo-scripts.postman_collection.json',
                                     '--environment', 'scripts/demoEnv.postman_environment.json',
                                     '--reporters', 'cli,htmlextra',
                                     '--reporter-htmlextra-export', 'target/newman/BooksService.html'])

                    archiveArtifacts allowEmptyArchive: true, artifacts: '**/target/newman/**'
                    //publish HTML Report
                    publishHTML([allowMissing: true, alwaysLinkToLastBuild: false, keepAll: true, reportFiles: 'BooksService.html', reportDir: 'target/newman/', reportName: 'API Tests-Books Service'])
                }
            }
        }

        stage('Security') {
            when {
                branch 'master'
            }
            steps {
                lock(resource: "${env.JOB_NAME}/30", inversePrecedence: true) {
                    milestone 30
                    echo '....Fortify scans'
                    fortifyExecuteScan(script: this, serverUrl: 'https://fortify.tools.sap/ssc',
                    buildTool: 'maven',
                    projectName: 'books-demo-service',
                    exclude: [ '**/target/**/*', '**/src/test/**/*', '**/gen/**/*' ],
                    buildDescriptorFile: 'srv/pom.xml',
                    dockerImage: 'docker.wdf.sap.corp:50000/piper/fortify:jdk11',
                    src: ['**/src/main/java/**/*'],
                    fortifyCredentialsId: 'fortifyToken')
                }
            }
        }

        stage('Promote to Nexus') {
            when {
                branch 'master'
            }

            steps {
                lock(resource: "${env.JOB_NAME}/35", inversePrecedence: true) {
                    milestone 35
                    echo 'Publish the MTAR to nexus'
                //   nexusUpload(script: this, url:'http://localhost:8081/repository/maven-releases/', nexusCredentialsId: 'nexusId' , groupId: 'demo-books-service', mavenRepository: 'maven-releases')
                }
            }
        }

        //Ready to release - Confirm to proceed
        stage('Confirm to Proceed') {
            when {
                branch 'master'
            }
            agent none
            options {
                timeout(time: 1, unit: 'HOURS')
            }

            steps {
                input message: 'Shall we proceed to production deployment?'
            }
        }

        stage('Prod - Deployment') {
            when {
                branch 'master'
            }

            steps {
                lock(resource: "${env.JOB_NAME}/40", inversePrecedence: true) {
                    milestone 40
                    echo '....Production Deployment - Cloud Foundry account'
                    cloudFoundryDeploy(script: this, apiEndpoint: 'https://api.cf.us10-001.hana.ondemand.com/', buildTool: 'mta', deployTool: 'mtaDeployPlugin', deployType: 'blue-green', space: 'prod', org: 'ac864addtrial', cfCredentialsId: 'cf_credential_id')

                    //healthcheck
                    script {
                        def checkUrl = 'https://demo-service-srv-prod.cfapps.us10-001.hana.ondemand.com/'
                        def statusCode = curl(checkUrl)
                        if (statusCode != '200') {
                            error "Health check failed: ${statusCode}"
                       } else {
                            echo "Health check for ${checkUrl} successful"
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            emailext(
        subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!',
        body: '$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS: Check console output at $BUILD_URL to view the results.',
        // recipientProviders: [[$class: 'DevelopersRecipientProvider']],
        to: 'ashsy009@gmail.com',
        /* groovylint-disable-next-line DuplicateStringLiteral */
        replyTo: 'ashsy009@gmail.com'
      )
        }
    }
}

def curl(url) {
    return sh(
    returnStdout: true,
    script: "curl -so /dev/null -w '%{response_code}' ${url}"
  ).trim()
}
