pipeline {
    agent any
    environment {
        VERSION = "${env.BUILD_ID}"
    }
    stages {
        stage("Sonar Quality Check") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'sonar-token') {
                        sh 'chmod +x gradlew'
                        sh './gradlew sonar'
                    }
                    timeout(time: 1, unit: 'HOURS') {
                        def qg = waitForQualityGate()
                        echo "Quality Gate Response: ${qg}" // Print the response for debugging
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }
        stage("docker build & docker push") {
            steps {
                script {
                    withCredentials([string(credentialsId: 'docker_pass', variable: 'docker_password')]) {
                        sh '''
                            docker build -t  34.229.254.193:8083/springapp:${VERSION} .
                            docker login -u admin -p $docker_password  34.229.254.193:8083
                            docker push  34.229.254.193:8083/springapp:${VERSION}
                            docker rmi  34.229.254.193:8083/springapp:${VERSION}
                        '''
                    }
                }
            }
        }
        stage('identifying misconfigs using datree in helm charts') {
            steps {
                script {
                    withEnv(["DATREE_TOKEN= .........."]){
                    dir('kubernetes/') {
                        sh 'helm datree test myapp/'
                    }
                    }
                }
            }
        }
        stage("pushing the helm chart to nexus") {
            steps {
                script {
                    
                    withCredentials([string(credentialsId: 'docker_pass', variable: 'docker_password')]) {
                      dir('kubernetes/') {
                        sh '''
                                 helmversion=$( helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' ')
                                 tar -czvf  myapp-${helmversion}.tgz myapp/
                                 curl -u admin:$docker_password http://34.201.164.31:8081/repository/helm-hosted/ --upload-file myapp-${helmversion}.tgz -v

                        '''
                    }
                    }
                }
            }
        }
    }
    post {
        always {
            mail bcc: '',
                body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL de build: ${env.BUILD_URL}",
                cc: '',
                charset: 'UTF-8',
                from: '',
                mimeType: 'text/html',
                replyTo: '',
                subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}",
                to: "bhanucorrect@gmail.com"
        }

    }
}
