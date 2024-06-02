pipeline {
    agent any

    environment {
        registry = "huzaifh02/vprofileapp"
        registryCredential = 'dockerhub'
        awsRegion = 'ap-south-1' // Update with your region
        eksClusterName = 'vprofile' // Update with your EKS cluster name
        helmRepo = 'https://github.com/huzaifh02/cicd-kube' // URL to your Helm charts repository
        helmChartPath = 'helm/vprofilecharts' // Path to the Helm chart within the repository
    }

    stages {
        stage('Setup Helm') {
            steps {
                sh '''
                helm version
                '''
            }
        }

        stage('Clone Helm Charts Repo') {
            steps {
                git branch: 'master', url: "${helmRepo}"
            }
        }

        stage('BUILD') {
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

        stage('UNIT TEST') {
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage('CODE ANALYSIS WITH CHECKSTYLE') {
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
            steps {
                script {
                    dockerImage = docker.build("${registry}:latest")
                }
            }
        }

        stage('Deploy Image') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("${env.BUILD_NUMBER}")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Remove Unused docker image') {
            steps {
                sh "docker rmi ${registry}:${env.BUILD_NUMBER}"
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
            }
        }

        stage('Wait for Quality Gate') {
            steps {
                timeout(time: 30, unit: 'MINUTES') {  // Increase timeout if necessary
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Deploy to EKS using Helm') {
            steps {
                script {
                    sh '''
                    helm upgrade --install myapp ${helmChartPath} --set image.repository=${registry} --set image.tag=${env.BUILD_NUMBER} --namespace default
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
