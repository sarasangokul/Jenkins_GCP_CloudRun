# Deploy App to Google Cloud Run using Jenkins CI/CD Pipeline
In this hands-on DevOps tutorial, we’ll walk through building a full-fledged Jenkins CI/CD pipeline to build, test, store, and deploy a Java application to Google Cloud Run using industry-standard tools and best practices.

## Requirements
1. Java:        Language & Framework 
2. Spring Boot: Creates Web Applications in Java
2. Maven:       Java Application Build Tool
3. Shell:       Commands to Build, Test, and Deploy
3. Jenkins:     CI/CD Pipeline Tool
4. Docker:      Containerization Tool
5. GCP Account & Google Cloud SDK


## Set up Jenkins with a Container
We want to build a docker image later with our Jenkins Pipeline, so we need a Jenkins Container or VM with Docker in it (i.e., Docker-in-Docker or DinD)

### Create a Docker Network
```sh
docker network ls
```
```sh
docker network create jenkins-java
```

```sh
docker run -d --name jenkins-dind \
-p 8080:8080 \
-p 50000:50000 \
-v /var/run/docker.sock:/var/run/docker.sock \
-v $(which docker):/usr/bin/docker \
-u root \
-e DOCKER_GID=$(getent group docker | cut -d: -f3) \
--network jenkins-java \
jenkins/jenkins:lts
```

### Check the Docker Processes Running
```sh
docker ps
```
Get Jenkins Container IP Address
```sh
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' jenkins-dind
```
On your browser, open
```sh
ContainerIP:8080
```

Note: If you see the error "It appears that your reverse proxy set up is broken.", go to Manage Jenkins > System and change the Jenkins URL from localhost:8080 to ContainerIP:8080.


### Log into the Jenkins Container
```sh
docker exec -it jenkins-dind bash
```

### Set up Docker in the Jenkins container
Inside the container, run the following commands: 
```sh
docker --version
```

```sh
groupadd -for -g $DOCKER_GID docker
usermod -aG docker jenkins
exit
```

```sh
docker ps
```

```sh
docker restart jenkins-dind
```

### Check the Logs of the Jenkins Container
This will also provide you the Jenkins Initial Password. Use the password to login to Jenkins & Install suggested plugins
```sh
docker logs -f jenkins-dind
```
Ctrl + C to exit logs


### Integrate Jenkins & GitHub Repository
In your GitHub Account, 
1. Create a PAT  with scope (repo & admin:repo_hook), and 
2. Add it as a Credential (Kind: Username with Psswd) on Jenkins UI
    1. Username:    GitHub Username here
    2. Password:    PAT here
3. Create Pipeline Job with Definition: "Pipeline script from SCM", and complete the fields SCM, Repository URL, Credentials, Branch Specifier, and Script Path. 
4. Apply and Save when done.


### Install Java & Maven in the Jenkins Container
```sh
docker exec -it jenkins-dind bash
```

Check Java version (Usually installed with Jenkins since Java is one of Jenkins' Dependencies) & JAVA_HOME
```sh
java -version
```
```sh
mvn -version
```

Install Maven & get MAVEN_HOME as well (usually: /usr/share/maven)
```sh
apt update -y
apt install maven -y
mvn -version
exit
```

***In the Jenkins GUI***
1. Install Maven Integration Plugin 
2. Set up Maven installation in Manage Jenkins > Tools > Maven installations - Uncheck "Install Automatically"
    1. Name:        maven387
    2. MAVEN_HOME:  /usr/share/maven
3. Set up JDK installation in Manage Jenkins > Tools > JDK installations - Uncheck "Install Automatically"
    1. Name:        java17015
    2. JAVA_HOME:   /opt/java/openjdk
4. Add these tools in the Jenkinsfile for 'jdk' and 'maven'.



## Set up SonarQube with a Container
Run the command below to create the SonarQube Container on the same Docker network as Jenkins:
```sh
docker run -d --name sonarqube-dind \
-e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
-p 9000:9000 \
--network jenkins-java \
sonarqube:latest
```

Check the logs of the SonarQube Container (Wait till SonarQube is Operational):
```sh
docker logs -f sonarqube-dind
```

### Check the Docker Processes Running
```sh
docker ps
```
Get SonarQube Container IP Address
```sh
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' sonarqube-dind
```
On your browser, open
```sh
ContainerIP:9000
```

Login to SonarQube GUI with 
```sh
Username: admin
Password: admin
```
and Change the password.


### Login to Jenkins Container & Establish Communication to the SonarQube Container
```sh
docker exec -it jenkins-dind bash
```
Update & Install Ping in Jenkins Container once you've logged in:
```sh
apt-get update -y
apt-get install iputils-ping -y
exit
```

***Use Jenkins Container bash to Ping SonarQube Container***
```sh
docker exec -it jenkins-dind ping sonarqube-dind
```
You will see bytes of data coming in showing established connection between the 2 containers.


### Create a SonarQube Token and Add it to Jenkins Credentials:
1. Click on the User Account, and click on "My Account"
2. Go to Security, create a token of type "Global Analysis Token", expiry date, Name: jmsonar, and Generate.
3. Copy and Save the token somewhere safe.
4. Go to Jenkins Credentials, select Kind "Secret text", Paste sonar token as the secret and provide ID and description.
5. Install the "SonarQube Scanner" and "Sonar Quality Gates" Plugins.
6. Go to Jenkins > Manage Jenkins > Systems > SonarQube Installation, Name: sonar, SonarQube URL (http://ContainerIP:9000), and Server auth token: select the credential. Apply and Save.
7. Go to Jenkins > Manage Jenkins > Tools > Add Name: sonar7, Install Automatically. Save and Apply


### Create Project in SonarQube
Create a Local Project in SonarQube and provide the ff:
1. Project display name, 
2. Project key, 
3. GitHub branch (main), 
4. Use the global setting, and 
5. Create project. 
6. Click on "Locally", Use existing token and enter name of your sonar token as value "jmsonar" and continue
7. Run analysis on your project: Select Other & Linux. Copy the execution script and add to your pipeline script. We will modify it a bit later.**
8. SonarQube Installation Name or SonarQubeEnv: sonar (the setup we did @ Manage Jenkins > System), add to pipeline
9. SonarQube Tool Name: sonar7 (the setup we did @ Manage Jenkins > Tools), add to pipeline as an environment variable.
10. Create a sonar-project.properties file in root of your GitHub Repository and add the SonarQube project key and organization (GitHub Username). 
11. Since the application is Java-based, you must specify the location of the Java binaries which is usually stored in a "target/classes" directory. Add it to the SonarScanner block as: "-Dsonar.java.binaries=target/classes". 


***To Generate Jenkins Pipeline Script***
1. In Jenkins UI, go to configure the Jenkins Pipeline
2. Click on Pipeline Syntax
3. Sample Step: Select withCredentials, Add Bindings, Select Secret text, Variable: sonarToken, Credentials: select the credentials 
4. Generate Pipeline Script.


## Trivy File System Scan
Login to the Jenkins Container
```sh
docker exec -it jenkins-dind bash
```
Check if Trivy is installed
```sh
trivy --version
```

If not, install Trivy in the Jenkins Container
```sh
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.17.2
```

```sh
trivy --version
exit
```
Add Trivy FS Scan stage to the Jenkinsfile


## Build Docker Image
Make sure Docker is installed in the Jenkins Container and there is a Dockerfile in the root of the GitHub Repo
```sh
docker --version
```

In Jenkins UI,
1. Install Docker & Docker Pipeline plugins in Jenkins UI
2. Manage Jenkins > Tools > Add Docker Installations: Name: Docker, Install Automatically from docker.com
3. Apply and Save.



## Trivy Docker Image Scan
Trivy is already set up. 
Now, scan the docker image with Trivy.


## Deployment to Google Cloud Platform (GCP)

### Push docker image to Artifact Registry & Deploy to Google Cloud Run
Go to your GCP Console home, 
1. Click on "APIs & Services
2. Search and Enable Artifact Registry API
3. Create a GCP Service Account:- Go to IAM & Admin > Service Accounts > Create service account > 
4. Permissions:- Search and Add Roles: Owner, continue
5. Principals with access:- Done
6. Generate a Json Key:- After creating the service account, click on the service account to open it and Click on "Keys" tab, click "Add Key", "Create new key", "JSON", and the key is downloaded onto your system (save if securely)
7. Go to "Artifact Registry" > Create repository: format=Docker, mode=standard, location type=region = us, desc=Docker repository, project=focal-dock-**


### Add GCP Credential to Jenkins
On Jenkins > Manage Jenkins > Credentials > Kind: Secret file, ID: gcp-jmsa, upload the .json key file


### Install Google Cloud SDK (gcloud) on Jenkins Container
Login to the Jenkins Container
```sh
docker exec -it jenkins-dind bash
```

```sh
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates gnupg
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | \
  sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | \
  sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
sudo apt-get update && sudo apt-get install -y google-cloud-sdk
```
Check that installation was successful:
```sh
gcloud version
exit
```


### Push Docker Image to GCR
In Jenkins > Pipeline Syntax > withCredentials > Bindings: Secret file, Variable: gcpCred, Credentials: select the gcp credential from earlier > Generate Pipeline Script. Copy script to Jenkinsfile & edit. 


### Deploy Java App Docker Image
Make sure that Cloud Run API is enabled. 
Run the Jenkins pipeline to deploy


### Setup Jenkins GitHub Webhooks
Automate Jenkins pipeline build on "git push"

1. On Jenkins, Go to the Jenkins job > Configure > Select the "GitHub hook trigger for GITScm polling" trigger > Apply & Save
2. On GitHub project repo > Settings > Webhooks > Add webhook 
3. But our Jenkins container is running locally thus GitHub can't locate its endpoint on the cloud. 
4. Since, I am not using a cloud VM, I will expose my Jenkins container to the cloud using NGROK bcuz Payload URL: the jenkins url (http://ip:8080/github-webhook/) will NOT work.
5. Install NGROK on your linux system (Ngrok creates a secure tunnel)
```sh
sudo snap install ngrok
```
6. Go to https://dashboard.ngrok.com/signup and sign up. Save your ngrok recovery files safe & setup MFA as well.
7. On your linux terminal, run the command that's given to add authtoken to the default ngrok.yml config file. The command looks like this: 
```sh
ngrok config add-authtoken someRandomStringHere
```
8. Now, Deploy your app online with the command
```sh
ngrok http http://ip:8080
```
9. You will get a public URL (forwarding link) that looks like 
```sh
https://582f-2603-8080-91f0-4ef0-e8ad-7ee-7ee8-4a4f.ngrok-free.app -> http://ip:8080
```

Now your payload URL should be:
```sh
Payload URL: the jenkins url (https://582f-2603-8080-91f0-4ef0-e8ad-7ee-7ee8-4a4f.ngrok-free.app/github-webhook/)
Content type: application/json
Secret: leave blank
SSL verification: Disable
Even: Just the push event
Active
Add webhook
```


## Clean Up
1. Delete the Cloud Run service
2. Delete the Artifact Registry Repo and Image
3. Delete the GCP Service Account
4. Stop and Delete Jenkins & SonarQube containers


# LIKE, COMMENT, & SUBSCRIBE To iQuant on YouTube !!!

