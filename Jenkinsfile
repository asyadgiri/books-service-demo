// #!/usr/bin/env groovy

// /*
//  * This file bootstraps the codified Continuous Delivery pipeline for extensions of SAP solutions such as SAP S/4HANA.
//  * The pipeline helps you to deliver software changes quickly and in a reliable manner.
//  * A suitable Jenkins instance is required to run the pipeline.
//  * More information on getting started with Continuous Delivery can be found here: https://sap.github.io/jenkins-library/
//  */

// @Library('piper-lib-os') _

// //piperPipeline script: this

/* groovylint-disable-next-line CompileStatic */
@Library('piper-lib-os') _   //to use the newman docker image

pipeline {
    agent any
    options {
        buildDiscarder logRotator(daysToKeepStr: '1', numToKeepStr: '3')
        skipDefaultCheckout()
    }
    stages {
        stage('Init') {
            steps {
                lock(resource: "${env.JOB_NAME}/5", inversePrecedence: true) {
                    milestone 5
                    deleteDir()
                    checkout scm
                }
            }
        }

        //Build the project via MBT Build Tool
        stage('Build') {
     
            steps {
                lock(resource: "${env.JOB_NAME}/10", inversePrecedence: true) {
                    milestone 10
                    mtaBuild(script: this,buildTarget: 'CF')
                stash name: 'deployment', includes: '*.mtar, mta.yaml'
                }
                  //  sh 'mbt build'
                }
            }
        }

        //Publish the Test Results
        stage('Publish Results') {
  
            steps {
                lock(resource: "${env.JOB_NAME}/15", inversePrecedence: true) {
                    milestone 15
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                    jacoco execPattern: '**/target/*.exec'
                    mavenExecuteStaticCodeChecks script: this
                }
            }
        }

          //Integration Stage
        stage('Integration Stage') {
               steps {
                lock(resource: "${env.JOB_NAME}/17", inversePrecedence: true) {
                    milestone 17
                    sh 'mvn clean install -s settings.xml'
                    junit allowEmptyResults: true, testResults:'**/integration-tests/target/surefire-reports/*.xml'
                    jacoco execPattern: '**/target/ITjacoco.exec'
                }
            }
        }

        


        //Ready to release - Confirm to proceed
        stage('Confirm to Proceed') {
            agent none
            options {
                timeout(time: 1, unit: 'HOURS')
            }
      
            steps {
                input message: 'Shall we proceed to production deployment?'
            }
        }

        stage('Prod - Deployment') {
  
            steps {
                lock(resource: "${env.JOB_NAME}/40", inversePrecedence: true) {
                    milestone 40
                    echo '....Production Deployment - Cloud Foundry account'
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
