pipeline {

    agent any

    tools {
        maven "3.9.7"
    }

    environment {
        registry = "ishfak00/nprofile-project"
        registryCredential = 'dockerhub'
        JAVA_HOME='/usr/lib/jvm/java-11-openjdk-amd64'
    }

    stages {

        stage('BUILD') {
            steps {
                sh 'mvn clean install -U'
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


        stage('Building App image') {
            steps{
              script {
                dockerImage = docker.build registry + ":V$BUILD_NUMBER"
              }
            }
        }
        
        stage('Deploy App Image') {
          steps{
            script {
              docker.withRegistry( '', registryCredential ) {
                dockerImage.push("V$BUILD_NUMBER")
                dockerImage.push('latest')
              }
            }
          }
        }

        stage('Remove Unused docker image') {
          steps{
            sh "docker rmi $registry:V$BUILD_NUMBER"
          }
        }

        stage('CODE ANALYSIS with SONARQUBE') {

            environment {
                scannerHome = tool 'mysonarscanner4'
            }

            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=nprofile-repo \
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
 
        stage('Gcloud Authntication') {
            agent { label 'JENKINS_AGENT' }
            environment {
                CLOUDSDK_CORE_PROJECT='primal-gear-411416'
                CLIENT_EMAIL='jenkins-user@primal-gear-411416.iam.gserviceaccount.com'
                GCLOUD_CREDS=credentials('gcloud-cred')
            }
            steps {
                sh '''
                    gcloud version
                    gcloud auth activate-service-account --key-file="$GCLOUD_CREDS"
                    gcloud container clusters get-credentials jenkins-cicd-gke-cluster-1 --zone us-east1-d --project primal-gear-411416
			        kubectl cluster-info
			        kubectl get nodes -o wide
                '''
            }
        }



        stage('Kubernetes Deploy') {
	        agent { label 'JENKINS_AGENT' }
                steps {
                    sh "helm upgrade --install -f nprofileapp-values.yaml nprofileapp-release helm/nproileapp --set vproapp.container.image=${registry}:V${BUILD_NUMBER} "
            }
        }

    }
}
