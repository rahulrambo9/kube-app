pipeline {

    agent any

	tools {
        maven "MAVEN3"
        jdk "OracleJDK8"
    }

    environment {
        registry = "rahulrambo9/vproappdock"
        registryCredential = 'dockerhub'
        ARTVERSION = "${env.BUILD_ID}"
    }

    stages{

        stage('BUILD'){
            steps {
                sh 'mvn clean install -DskipTests=true'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {

            environment {
                scannerHome = tool 'mysonarscanner4'
            }

            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build App Image'){
           steps{
              script {
                  dockerImage = docker.build registry + ":V$BUILD_NUMBER"
              }
           }

        }

        stage('upload image'){
           steps{
              script {
                  docker.withRegistry('', registryCredential) {
                    dockerImage.push("V$BUILD_NUMBER")
                    dockerImage.push('latest')
                  }
              }
           }
        }

        stage('Remove unused dockerIMG'){
           steps {
              sh "docker rmi $registry:V$BUILD_NUMBER"
           }
        }

        stage('kubernetes Deploy'){
           agent {label 'KOPS'}
           steps {
             sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
           }
        }

    }


}