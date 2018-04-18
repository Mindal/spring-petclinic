pipeline {
    agent any
    environment {
        EMAIL_RECIPIENTS = 'a.bogdanov1993@gmail.com'
    }
    stages {

        stage('Build with unit testing') {
            steps {
                // Run the maven build
                script {
                    // Get the Maven tool.
                    // ** NOTE: This 'M3' Maven tool must be configured
                    // **       in the global configuration.
                    echo 'Pulling...' + env.BRANCH_NAME
                    def mvnHome = tool 'localMaven'
                    bat(/"${mvnHome}\bin\mvn" -Dintegration-tests.skip=true clean package/)
                    archive 'target*//*.war'
                }

            }
        }
        stage('Sonar scan execution') {
            // Run the sonar scan
            steps {
                script {
                    def mvnHome = tool 'localMaven'
                    withSonarQubeEnv {

                        bat (/"${mvnHome}\bin\mvn verify sonar:sonar/)
                    }
                }
            }
        }
        // waiting for sonar results based into the configured web hook in Sonar server which push the status back to jenkins
        stage('Sonar scan result check') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    retry(3) {
                        script {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                error "Pipeline aborted due to quality gate failure: ${qg.status}"
                            }
                        }
                    }
                }
            }
        }
        stage('Development deploy approval and deployment') {
            steps {
                script {
                    if (currentBuild.result == null || currentBuild.result == 'SUCCESS') {
                        timeout(time: 3, unit: 'MINUTES') {
                            // you can use the commented line if u have specific user group who CAN ONLY approve
                            //input message:'Approve deployment?', submitter: 'it-ops'
                            input message: 'Approve deployment?'
                        }
                        timeout(time: 2, unit: 'MINUTES') {
                            //
                            if (developmentArtifactVersion != null && !developmentArtifactVersion.isEmpty()) {
                                // replace it with your application name or make it easily loaded from pom.xml
                                def jarName = "application-${developmentArtifactVersion}.jar"
                                echo "the application is deploying ${jarName}"
                                // NOTE : CREATE your deployemnt JOB, where it can take parameters whoch is the jar name to fetch from jenkins workspace
                                build job: 'ApplicationToDev', parameters: [[$class: 'StringParameterValue', name: 'jarName', value: jarName]]
                                echo 'the application is deployed !'
                            } else {
                                error 'the application is not  deployed as development version is null!'
                            }

                        }
                    }
                }
            }
        }
    }
    post {
        // Always runs. And it runs before any of the other post conditions.
        always {
            // Let's wipe out the workspace before we finish!
            deleteDir()
        }
        success {
            sendEmail("Successful");
        }
        unstable {
            sendEmail("Unstable");
        }
        failure {
            sendEmail("Failed");
        }
    }


}

def sendEmail(status) {
    mail(
            to: "$EMAIL_RECIPIENTS",
            subject: "Build $BUILD_NUMBER - " + status + " (${currentBuild.fullDisplayName})",
            body: "\n\n Check console output at: $BUILD_URL/console" + "\n")
}


