Flow of the CICD:  
                                --> Docker --> Nexus Repository  
Dev Team --> GitHub --> Jenkins --> Gradle --> Sonarqube  
                                --> Helm --> Datree --> Kubernetes  


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





