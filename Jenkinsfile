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
                            docker build -t 54.237.103.169:8083/springapp:${VERSION} .
                            docker login -u admin -p $docker_password 54.237.103.169:8083
                            docker push 54.237.103.169:8083/springapp:${VERSION}
                            docker rmi 54.237.103.169:8083/springapp:${VERSION}
                        '''
                    }
                }
            }
        }
    }
}

