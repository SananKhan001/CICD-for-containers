pipeline {

    agent any

	tools {
        maven "MAVEN3"
        jdk "OracleJDK17"
    }

    environment {
        registry = 'sanan3946/vprofile'
        registryCredential = 'dockerhub'
        SONARSCANNER = 'sonarscanner'
        SONARSERVER = 'sonar-prod'
    }

    stages{

        stage('BUILD'){
            steps {
                sh 'mvn clean install -DskipTests'
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


        stage('Building image') {
            steps{
              script {
                dockerImage = docker.build registry + ":v$BUILD_NUMBER"
              }
            }
        }
        
        stage('Deploy Image') {
          steps{
            script {
              docker.withRegistry( '', registryCredential ) {
                dockerImage.push("v$BUILD_NUMBER")
                dockerImage.push('latest')
              }
            }
          }
        }

        stage('Remove Unused docker image') {
          steps{
            sh "docker rmi $registry:v$BUILD_NUMBER"
          }
        }

        stage('Sonar Analysis') {
            environment {
                scannerHome = tool "${SONARSCANNER}"
            }
            steps {
                withSonarQubeEnv("${SONARSERVER}") {
                   sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                } 
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(2) {
                    // some block 
                    waitForQualityGate abortPipeline: true 
                }
            }
        }
        stage('Kubernetes Deploy') {
            agent { label 'KOPS' }
            steps {
                    sh "helm upgrade --install --force vproifle-stack helm/vprofilecharts --set appimage=${registry}:v${BUILD_NUMBER} --namespace prod"
            }
        }

    }


}
