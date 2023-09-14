pipeline{
    agent any
    stages{
        stage("Sonar Quality Check"){
            agent{
                docker {
                    image 'openjdk:11'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps{
                script{
                    withSonarQubeEnv(credentialsId: 'sonar-token') {
                             sh 'chmod +x gradlew'
                             sh './gradlew sonarqube'
               }
            }
        }
        }
    }
}