Flow of the CICD:  
----------------------->Jenkins --> Docker --> Nexus Repository  
Dev Team --> GitHub --> Jenkins --> Gradle --> Sonarqube  
----------------------->Jenkins--> Helm --> Datree --> Kubernetes  


Github to store application code, Dockerfile, Jenkinsfile, kubernetes manifest files.  
Gradle Sonarqube plugin is used for static code analysis and gradle is also used for building the application.  
Sonarqube used for static code analysis( a method of debugging a product by examining its source code before it's run.) Validate against sonar rules.  
Jenkins: Opensource continuous integration server written in JAVA. In this project it is used for integrating all the other tools github, gradle, sonarqube, docker, nexus, kubernetes, datree.  
Docker: Writing multi-stage dockerfile to containerize the application, the image is pushed to nexus repository.  
Nexus: It is a repository manager. In the project it is used to store docker images and helm charts.(Private registry)  
Kubernetes: Deploying our application on kubernetes using helm charts.  
Helm: Helps us manage the k8s applications using helm charts. Say you have to make change to each manifest file, with helm you can just make that change in values.yml. Similar to this their are many other benefits.  
Datree: Prevents kubernetes misconfigurations from reaching production.  


Steps:  
1.Pull the code from github  
2.Do the static code analysis using Sonarqube with the help of sonarqube gradle plugin  
3.Check the status of quality gate in sonar  
4.Using the multistage dockerfile build the code, generate artifact and create docker image   
5.Push the image to the private repository which is in nexus  
6.Check if any misconfigurations in helm charts  
7.Helm charts push to nexus  
8.Manual approval  
9.Deploy to k8s cluster  
10.If any stage fails we will send an email  

5 ec2 machines(t2-medium) ubuntu  
  
Installations:  java -version
1. Jenkins 2. Sonarqube 3. Nexus 4. k8s-master 5. k8s-node

1.Jenkins Server:  
  sudo apt update  
  sudo apt install openjdk-11-jdk  
  java -version  
  Install jenkins:  
  https://www.jenkins.io/doc/book/installing/linux/  
  Install helm:
  curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3  
  chmod 700 get_helm.sh  
  ./get_helm.sh  
  Install datree plugin:  
  https://github.com/datreeio/helm-datree
  Install docker:  https://github.com/datreeio/helm-datree  
ip:8080

2. Sonarqube
   Install Docker: https://docs.docker.com/engine/install/ubuntu/
   docker run -d -p 9000:9000 sonarqube:lts
   ip:9000

3. Nexus
   apt-get install wget ( install if you dont have wget )  
   java -version ( make sure java is installed which should be java 8 or higher version )  
   wget https://download.sonatype.com/nexus/3/latest-unix.tar.gz   
   tar -xvf latest-unix.tar.gz    
   cd nexus-3.35.0-02/bin  
   ./nexus start ( starts the nexus artifactory )  
   ./nexus status ( by this you check the status of nexus artifactory )  
   To access this use http://ip_Address:8081 
   admin:admin  

4. Setup K8s
   https://medium.com/@mehmetodabashi/installing-kubernetes-on-ubuntu-20-04-e49c43c63d0c  
   Command which didn't worked in the article above:
   ```
   sudo tee /etc/docker/daemon.json <<EOF  
{  
  "exec-opts": ["native.cgroupdriver=systemd"],  
  "log-driver": "json-file",  
  "log-opts": {  
    "max-size": "100m"  
  },  
  "storage-driver": "overlay2"  
}  

EOF  
```
An error: https://programmerall.com/article/99842435629/
________________________________________________________________________________________________________________________________________________________________________________________________________________

For building the application we use ./gradlew build and for static code analysis we use ./gradlew sonar  

We will create jenkins pipeline  
New Item > Gave repository url > */devops branch selected  

Clone the repo with the src code in local and let us start writing the jenkinsfile   (Install few extensions related to jenkinsfile in vscode)  

Stage1:  sonar quality check  
Install plugins in jenkins:  Sonarqube scanner, sonar gerrit, sonarqube generic coverage, sonarquality gate, quality gate.Install docker related plugins too if you want to use docker as an agent.  
Configure System > go to sonarqube section > save the url and also create a credential in jenkins where you save the sonarqube token which you then select in configure system section.  

```
pipeline {  
    agent any  
    stages {
        stage("Sonar Quality Check") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'sonar-token') {
                        sh 'chmod +x gradlew'
                        sh './gradlew sonar'
                    }
                }
            }
        }
    }
}
```

Push this code to devops branch and test by running the pipeline in jenkins.

Now we need to check the status of Quality Gates, if it is okay or not.

On sonarqube server > Administration > Configuration > Webhook (Create a webhook here). Now we will add the part to check quality gate status in our jenkinsfile.


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
    }
}  




