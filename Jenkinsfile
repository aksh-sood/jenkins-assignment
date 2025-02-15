pipeline {
    agent any

    tools {
        maven "MAVEN3.9"
        jdk "JDK21"
    }

    parameters {
        booleanParam(name: 'SONARQUBE_SCAN', defaultValue: true, description: 'Run SonarQube Scan?')
        booleanParam(name: 'RUN_LIZARD', defaultValue: true, description: 'Run Lizard Complexity Analysis?')
        booleanParam(name: 'RUN_DP_CHECK', defaultValue: true, description: 'Run Dependency Security Check?')
        string(name: 'ALLOWED_CRITICAL_VUNERABILITIES', defaultValue: '4', description: 'Fail on Critical Vulnerabilities Threshold')
    }

    environment {
        jobName = "${env.JOB_NAME}".tokenize('/').last()
        scannerHome = tool 'sonar'
    }

    stages {


        stage('Build') {
            steps {
                script {
                    try {
                        sh "mvn install -DskipTests"
                    } catch (Exception e) {
                        error "❌ Build failed: ${e.message}"
                    }
                }
            }
            post {
                success {
                    echo "✅ Archiving artifacts"
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

        stage('Test') {
            steps {
                script {
                    try {
                        sh "mvn test"
                    } catch (Exception e) {
                        unstable "⚠️ Tests failed: ${e.message}"
                    }
                }
            }
        }

        stage("SonarQube Analysis & Quality Gate") {
            when {
                expression { return params.SONARQUBE_SCAN }
            }
            steps {
                script {
                    try {
                        withSonarQubeEnv('sonarserver') {
                            sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                                -Dsonar.projectName=vprofile-repo \
                                -Dsonar.projectVersion=1.0 \
                                -Dsonar.sources=src/ \
                                -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                                -Dsonar.junit.reportsPath=target/surefire-reports/ \
                                -Dsonar.jacoco.reportsPath=target/jacoco.exec '''
                        }
                    } catch (Exception e) {
                        error "❌ SonarQube analysis failed: ${e.message}"
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

        stage('Cyclomatic Complexity') {
            when {
                expression { return params.RUN_LIZARD }
            }
            steps {
                script {
                    try {
                        sh(script: "lizard src/main/java/ > lizard-${jobName}.txt", returnStatus: true)
                    } catch (Exception e) {
                        unstable "⚠️ Lizard complexity analysis failed: ${e.message}"
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "lizard-${jobName}.txt", fingerprint: true
                }
            }
        }

        stage("Security Scan") {
            when {
                expression { return params.RUN_DP_CHECK }
            }
            steps {
                script {
                    try {
                        def result = dependencyCheck(
                            additionalArguments: "--format XML --nvdApiKey ${env.DP_API_KEY} --out dependency-check-report-${jobName}.xml",
                            odcInstallation: 'DP-Check',
                            returnStatus: true
                        )

                        dependencyCheckPublisher(
                            pattern: "**/dependency-check-report-${jobName}.xml",
                            failedTotalCritical: params.ALLOWED_CRITICAL_VUNERABILITIES
                        )

                        if (currentBuild.result == 'UNSTABLE') {
                            unstable('⚠️ Dependency check found vulnerabilities')
                        } else if (currentBuild.result == 'FAILURE') {
                            error('❌ Dependency check failed due to critical vulnerabilities')
                        }
                    } catch (Exception e) {
                        error "❌ Security Scan failed: ${e.message}"
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "dependency-check-report-${jobName}.xml", fingerprint: true
                }
            }
        }

        stage("Upload Artifact") {
            steps {
                script {
                    try {
                        nexusArtifactUploader(
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            nexusUrl: "${env.NEXUS_URL}:8081",
                            groupId: 'QA',
                            version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                            repository: 'vprofile-repo',
                            credentialsId: 'nexus',
                            artifacts: [[
                                artifactId: 'vproapp',
                                classifier: '',
                                file: 'target/vprofile-v2.war',
                                type: 'war'
                            ]]
                        )
                    } catch (Exception e) {
                        error "❌ Artifact upload failed: ${e.message}"
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                def message = "✅ Successfully generated ${jobName}\nVersion: ${BUILD_ID}\n🔗 Build URL: ${env.BUILD_URL}"
                slackSend(channel: "#jenkins", color: '#00FF00', message: message)
            }
        }
        
        unstable {
        script {
            def message = "⚠️ The ${jobName} build is UNSTABLE.\nVersion: ${BUILD_ID}\n🔗 Build URL: ${env.BUILD_URL}"
            slackSend(channel: "#jenkins", color: '#FFFF00', message: message)
            }
        }
        
        failure {
            script {
                def message = "❌ The ${jobName} generation failed.\nVersion: ${BUILD_ID}\n🔗 Build URL: ${env.BUILD_URL}"
                slackSend(channel: "#jenkins", color: '#FF0000', message: message)
            }
        }
    }
}

