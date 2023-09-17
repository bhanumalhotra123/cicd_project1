Flow of the CICD:  
----------------------->Jenkins --> Docker --> Nexus Repository  
Dev Team --> GitHub --> Jenkins --> Gradle --> Sonarqube  
----------------------->Jenkins--> Helm --> Datree --> Kubernetes  
  


Overview of tech used:  

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

![1](https://github.com/bhanumalhotra123/cicd_project1/assets/144083659/170229f9-4c72-46cb-ad6c-867be38c21b1)

Now we need to check the status of Quality Gates, if it is okay or not.

![ac](https://github.com/bhanumalhotra123/cicd_project1/assets/144083659/c3a46a2b-e4cb-4ccc-9110-77850348394f)

On sonarqube server > Administration > Configuration > Webhook (Create a webhook here). Now we will add the part to check quality gate status in our jenkinsfile.

   
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
```

    
Now the stage to create dockerfile and push the image to repository in nexus.  
First we need to create a docker-hosted repository.  
Nexus Server > Create Repository > docker(hosted) > give the name and http port as 8083   
![4](https://github.com/bhanumalhotra123/cicd_project1/assets/144083659/974ff9e6-ec00-48f6-8543-ba411be35307)  

  
On jenkins host also we need to configure this repository  
To setup Insecure Registries: we need to edit or if not present create a file /etc/docker/daemon.json in that file add details of nexus  

{ "insecure-registries":["nexus_machine_ip:8083"] }

once that's done we need to execute systemctl restart docker this is to apply new changes, also we can verify whether registry is added or not by executing  
docker info   
  
once this is done from jenkins host you can try docker login -u nexus_username -p nexus_pass nexus_ip:8083  
![6](https://github.com/bhanumalhotra123/cicd_project1/assets/144083659/06df03a2-83c0-422d-a1cf-4d9089585cdf)



Now Dockerfile: 

```
FROM openjdk:11 as base
WORKDIR /app
COPY . .
RUN chmod +x gradlew
RUN ./gradlew build 

FROM tomcat:9
WORKDIR webapps
COPY --from=base /app/build/libs/sampleWeb-0.0.1-SNAPSHOT.war .
RUN rm -rf ROOT && mv sampleWeb-0.0.1-SNAPSHOT.war ROOT.war
```

We will create a secret text in jenkins in which we save docker password as docker_pass



```
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
                            docker build -t 34.229.254.193:8083/springapp:${VERSION} .
                            docker login -u admin -p $docker_password 34.229.254.193:8083
                            docker push 34.229.254.193:8083/springapp:${VERSION}
                            docker rmi 34.229.254.193:8083/springapp:${VERSION}
                        '''
                    }
                }
            }
        }
    }
}
```


![7](https://github.com/bhanumalhotra123/cicd_project1/assets/144083659/f2b1bbcf-8e9c-492f-bdf3-ae1f9ff6f56f)

  


Now for sending the email about success or failure, install the email extension plugin in jenkins and configure the settings for it under Manage Jenkins > Configure System. We will be using smtp.gmail.com 


After this we add the code to jenkinsfile for sending an email.
  

```
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
                            docker build -t 34.229.254.193:8083/springapp:${VERSION} .
                            docker login -u admin -p $docker_password 34.229.254.193:8083
                            docker push 34.229.254.193:8083/springapp:${VERSION}
                            docker rmi 34.229.254.193:8083/springapp:${VERSION}
                        '''
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
```







  




