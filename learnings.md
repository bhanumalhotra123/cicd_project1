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


