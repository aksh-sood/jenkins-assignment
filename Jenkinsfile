pipeline {
    agent any

    tools {
        maven "MAVEN3.9"
        jdk "JDK21"
    }

    stages {
        
                        stage ('Build') {
steps {
    sh "mvn install -DskipTests"
}
post {
    success{
        echo "Archiving artifact"
        
        jacoco(
                    execPattern: 'target/jacoco.exec', 
                    classPattern: 'target/classes', 
                    sourcePattern: 'src/main/java', 
                    exclusionPattern: '**/test/**'
                )
        archiveArtifacts artifacts: '**/*.war'
    }
}
        }

                stage ('Test') {
steps {
    sh "mvn test"
}
        }


stage("SonarQube Analysis & Quality Gate") {
    when {
        expression { return params.SONARQUBE_SCAN == true }
    }
    environment {
        scannerHome = tool 'sonar'
    }
    steps {
        script {
            withSonarQubeEnv('sonarserver') {
                sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                       -Dsonar.projectName=vprofile-repo \
                       -Dsonar.projectVersion=1.0 \
                       -Dsonar.sources=src/ \
                       -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                       -Dsonar.junit.reportsPath=target/surefire-reports/ \
                       -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                       -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
            }
        }
    }
    post {
        success {
            script {
                timeout(time: 1, unit: 'HOURS') {
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        error "❌ SonarQube Quality Gate failed!"
                    }
                }
            }
        }
    }
}


                        stage ('Cyclomatic Complexity') {
                               when {
        expression { return params.RUN_LIZARD == true }
    }
steps {
        script {
            def jobName = JOB_NAME.tokenize('/').last()
            sh(script: "lizard src/ > lizard-${jobName}.txt", returnStatus: true)

        }
}
    post {
        always {
            script{
            // Archive Lizard report
            def jobName = JOB_NAME.tokenize('/').last()
            archiveArtifacts artifacts: "lizard-${jobName}.txt", fingerprint: true
            }
        }
    }
        }


stage("Security Scan") {
        when {
        expression { return params.RUN_DP_CHECK }
    }
    steps {
        script {
              def jobName = JOB_NAME.tokenize('/').last()
            def result = dependencyCheck(
                additionalArguments: "--format XML --nvdApiKey ${env.DP_API_KEY} --out dependency-check-report-${jobName}.xml ",
                odcInstallation: 'DP-Check',
                returnStatus: true
            )

dependencyCheckPublisher (
            pattern: "**/dependency-check-report-${jobName}.xml",
            // failedTotalLow: 1,
            // failedTotalMedium: 1,
            // failedTotalHigh: 1,
            failedTotalCritical: params.DP_CHECK_LEVEL
          )
                    if (currentBuild.result == 'UNSTABLE') {
            unstable('UNSTABLE: Dependency check')
          } else if (currentBuild.result == 'FAILURE') {
            error('FAILED: Dependency check')
          }
        }
    }
    post {
        always {
            script{
              def jobName = JOB_NAME.tokenize('/').last()
            archiveArtifacts artifacts: "dependency-check-report-${jobName}.xml", fingerprint: true
            }
        }
    }
}

stage("upload artifact"){
    steps{
nexusArtifactUploader(
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            nexusUrl: "${env.NEXUS_URL}:8081",
                            groupId: 'QA',
                            version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                            repository: 'vprofile-repo',
                            credentialsId: 'nexus',
                            artifacts: [
                                [artifactId: 'vproapp',
                                classifier: '',
                                file: 'target/vprofile-v2.war',
                                type: 'war']
                            ]
                        );

    }
}
    }

    
    post {
        success { 
            script{
                def jobName = JOB_NAME.tokenize('/').last()
                def message = "Successfully generated the ${jobName}\nVersion: "+ BUILD_ID +"\nBuild URL: ${env.BUILD_URL}"
                slackSend (channel: "#jenkins", color: '#00FF00', message: message)
            }
        }
        failure {          
            script{
                def jobName = JOB_NAME.tokenize('/').last()
                def message = "The ${jobName} generation failed.\nVersion: "+ BUILD_ID +"\nBuild URL: ${env.BUILD_URL}"
                slackSend (channel: "#jenkins", color: '#FF0000', message: message)
            }
        }
    }
    
}
