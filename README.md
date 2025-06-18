# Objective

The purpose of this project is to design and implement a CI/CD pipeline using Jenkins to automate the deployment of an Ecommerce website. 

## Installing and Configuring Jenkins on our Web Server

The first step in this process is to configure Jenkins on our EC2 instance. 

Our Web server is an AWS EC2 instance, instance type is a t3.medium and our AMI image is Ubuntu.

Here are the following commands we ran to install Jenkins on this instance 

- Update the system and updates package repositories.

![alt text](<img/img 1..png>)




- Installing Java 

![alt text](<img/img 2..png>)

*Jenkins requires Java tor run. There are mulitple Java implementations, OpenJDK is the most popular at the moment, however, we need a specific Java version that is compatible with Jenkins*

*To find out which Java version is compatible, you can run the following command*

![alt text](<img/img 3..png>) 

*Java 17 or 21 is recommended, rather than Java 11, because Java 11 is gradually being phased out and some plugins may not function properly if Jenkins is using Java 11*


- Verify which Java version is installed by running this command

![alt text](<img/img 4..png>)

- the following series of command installs jenkins (long term support release) on Debian based distributions like Ubuntu. https://www.jenkins.io/doc/book/installing/linux/ 


        sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
        https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
        echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
        https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
        /etc/apt/sources.list.d/jenkins.list > /dev/null
        sudo apt-get update
        sudo apt-get install jenkins

- To start and enable Jenkins, run the following commands

![alt text](<img/img 6..png>)

### Launching Jenkins

After installing Jenkins, we can launch Jenkins using our EC2 instance IPv4 address combined with the port for Jenkins.
By default, Jenkins listens to port 8080. hence, an inbound rule must be created on our instance security group that allows for inbound traffic on port 8080

        54.167.81.194:8080

Jenkins immediately request for an Administrator password. This can be retrieved from the `initialAdminPassword` file.

![alt text](<img/img 7..png>)

![alt text](<img/img 8..png>)

-  Install Suggested plugins and create a first Admin user and sign in to the Jenkins console

![alt text](<img/img 9..png>)

*Note that the `initalAdminPassword` is only available when you **initially** sign in to Jenkins. 
Once Jenkins is fully comfigured, credentials are stored elsewhere and `initialAdminPassword` would not work during sign in because that file would not be available*  

**Signing back in to Jenkins** 

There are some reconfiguration steps in other to log back in to Jenkins without an inital Admin Password. 
    

<ins>How to log back in to Jenkins 
    
Open the configure.xml file 
    
`$ sudo vim config.xml`
    
In the configuration file, remove authentication 
    
        ○ Find the line 
            § <useSecurity>true</useSecurity>
    
        ○ Change it to 
        
            § <useSecurity>false</useSecurity>
    
    
        ○ Restart Jenkins 
            § sudo systemctl start jenkins
    
After restarting, Jenkins allow access without authentication. 

Its is not recommended to sign in to Jenkins without authentication, hence, you should create a new credential for signing, this is done under "Manage Jenkins" and "configure credentials"

![alt text](<img/img 10..png>)

## Git Repository Integration

Once Jenkins is successfully installed, we need to integrate Jenkins with our version control system for source code management (Git).

<ins>Jenkins Freestyle Job 

To do this, we first creat a New Item in Jenkins, we called it freestyle_project, and choose the item type as Freestyle Project. 

![alt text](<img/img 22..png>)

*Notice the name we have given this item, there is no space in the project name, it is separated an underscore(__). We must avoid spaces when naming projects because it can cause issues with script execution, and automation process. In naming conventions, underscores (_) or hyphens (-) are often used*

<ins> Instruction for Jenkins 

By clicking Configure on our newly created project, we setup instructions for Jenkins to trigger automated actions whenever certain changes (i.e code changes) occur in our github repository. 

We have specified our git repository URL in this section.

![alt text](<img/img 29..png>)

![alt text](<img/img 23..png>)

*Note that we have used the repository URL that end with .git in the source code management, this is different from the website URL.*

We have our Branch as `main`, same as our git repository.  


We've selected our Build trigger as "Github hook trigger for GITScm polling" which allows builds to be triggered automatically when changes occur in the repository. Github sends a webhook notification to Jenkins whenever a commit/change is made in the repostory.

![alt text](<img/img 25..png>)





<ins> Configuring Webhooks on Git 

In our github repository, we can configure a webhook that would notify an external service whenever there are any code changes. 

To configure this we simply need to navigate to settings in our repository and select Webhooks. Click **Add Webhook** and set the **Payload URL** to our Jenkins server.  URL followed by `/github-webhook/` 

    http://54.167.81.194:8080/github-webhook/

![alt text](<img/img 26..png>)


Once this configuration is done, we ran a test by making a change in our README.MD file in github repository.

We see below  that as soon as we made a change, a new build was launched automatically. 
![alt text](<img/img 27..png>)


## Installing Git and Docker on our Web-sever

### Installing Git
A key component in designing a Continuous Integration and Continuous Delivery (CI/CD) pipeline is integrating a version control system like Git. 

By tightly linking Jenkins with Git, developers can trigger automated builds that can run, test and deploy an application. 

This ensures that  code changes are continuously integrated and tested, allowing for faster and more reliable software delivery.  

The following command were run to install Git on our web-server 

- Update package lists to ensure we are using the latest version

![alt text](<img/img 11..png>)

- Install Git using the package manager
![alt text](<img/img 12..png>)

- Verify the installation by checking the Git version
![alt text](<img/img 13..png>)


### Installing Docker 

Our Ecommerce application would run on a docker container. We installed docker on the same instance jenkins was installed using shell script execution for installation.   

Here are the steps 
- Create a file named docker.sh 

![alt text](<img/img 14..png>)

- Open the bash shell file and paste the script below 

![alt text](<img/img 15..png>)

- Make the file executable 

![alt text](<img/img 16..png>)


- Execute the file 

![alt text](<img/img 17..png>)

After installing Docker, the next step is to create a Dockerfile, which will contain the application and define its environment setup. Our application will be hosted on an Nginx web server running on port 80.

![alt text](<img/img 18..png>)


![alt text](<img/img 19..png>)

**Application - index.html** 

![alt text](<img/img 20..png>)
![alt text](<img/img 21..png>)




*A vital step in Docker installation process is to add Jenkins User to the Docker group
This would make sure that Jenkins has the necessary permission  that is needed to connect Docker, hence preventing any permission denied error output you receive trying to connect to the Docker daemon socket.* 

![alt text](<img/img 35..png>)



### Building a Pipeline Script on Jenkins
 

Let build a Jenkins pipeline script that automatates a connection to our source code management repository (Github) and at the same time explicitly build a docker image `dockerfile`, pushes the image to a our docker registry `dara433/my-web-server` and runs the docker container.




        pipeline {
            agent any
        
            environment {
                IMAGE_NAME = 'dockerfile'
                DOCKERHUB_REPO = 'dara433/my-web-server'
                IMAGE_TAG = 'latest'
                DOCKER_CREDENTIALS_ID = 'docker-hub-credentials'
            }
        
            stages {
                stage('Clone Repository') {
                    steps {
                        checkout scmGit(
                            branches: [[name: '*/main']],
                            extensions: [],
                            userRemoteConfigs: [[
                                credentialsId: '21dc4437-f35e-4308-8408-86cba584d730',
                                url: 'https://github.com/Dara433/Automating-deployment-of-an-Ecommerce-Website.git'
                            ]]
                        )
                    }
                }
        
                stage('Build Docker Image') {
                    steps {
                        sh "docker build -t ${IMAGE_NAME} ."
                    }
                }
        
                stage('Tag and Push Docker Image') {
                    steps {
                        withCredentials([usernamePassword(
                            credentialsId: "${DOCKER_CREDENTIALS_ID}",
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )]) {
                            script {
                                sh "docker tag ${IMAGE_NAME} ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                                sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                                sh "docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                                sh "docker logout"
                            }
                        }
                    }
                }
        
                stage('Run Docker Container') {
                    steps {
                        sh "docker run -d -p 8081:80 ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                    }
                }
            }
        }
        
        


**Explanation of the script**

    agent any 

Specifiies that the pipeline can run on any available agent (an agent can be either a jenkins master or node). This means the pipeline is not tied to a specific node

    Environment {

Specifies all environment variables that is going to be used in phases of the delivery process. 


We begin to define the various stages of the pipeline, each representing a phase in the software delivery process. 

    stages {



<ins> Stage 1: Connect to Github and Clone the repository

    stage('Clone Repository') {
                    steps {
                        checkout scmGit(
                            branches: [[name: '*/main']],
                            extensions: [],
                            userRemoteConfigs: [[
                                credentialsId: '21dc4437-f35e-4308-8408-86cba584d730',
                                url: 'https://github.com/Dara433/Automating-deployment-of-an-Ecommerce-Website.git'
                            ]]
                        )
                    }


            
This stage checks out the Source code (main branch) from the Github repository in the Jenkins workspace. We used a pipeline syntax checkoutSCM that authenticates Jenkins credentials and clones our Github repository. 
The snippet generator is a helpful tool for generating this script.

![alt text](<img/img 30..png>)
![alt text](<img/img 31..png>)


<ins>Stage 2: Build Docker Image 

            
            stage('Build Docker Image') {
                    steps {
                        sh "docker build -t ${IMAGE_NAME} ."
                    }
                }


This stage builds a Docker image `dockerfile` using the source code obtained from the Github repository. The `docker build` command is executed using the shell (`sh`)



<ins> Stage 3: Tag and Push Docker Image

            stage('Tag and Push Docker Image') {
                    steps {
                        withCredentials([usernamePassword(
                            credentialsId: "${DOCKER_CREDENTIALS_ID}",
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )]) {
                            script {
                                sh "docker tag ${IMAGE_NAME} ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                                sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                                sh "docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                                sh "docker logout"



This step pushes the docker image to the docker registry.


Here, we need our Docker credentials to authenticate us into the docker registry. 
Stored credentials are managed by going to Jenkins Dashboard - Manage Jenkins - Credentials. We added our Username and Password for docker, ID is named as  `docker-hub-credentials` 


*Note `docker-hub-credentials` is one of our environment variable in the pipeline script, and it has been assigned as DOCKER_CREDENTIALS_ID* 

![alt text](<img/img 32..png>)

            usernameVariable: 'DOCKER_USER', - represents the Docker Hub username
            passwordVariable: 'DOCKER_PASS'   - represents the Docker Hub password 

These are temporary envoirnment varibles that get injected only inside the `withCredentials` block from our stored credentials.

The script securely pull these credentials in the `withCredentials` block, without having to hardcode them in the script, it keeps the username and password hidden from build logs making our Jenkins piepline more secure. 



To push Docker images to an existing repository, we followed these steps in the shell execution (`sh`).

We begin by tagging the image 

        sh "docker tag ${IMAGE_NAME} ${DOCKERHUB_REPO}:${IMAGE_TAG}"

logging in to the Docker terminal
 
        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"

- This shell command tells Docker to:
    - Use the provided username (-u $DOCKER_USER).
    - Accept the password from the standard input (--password-stdin), which comes from the echo part.

 It avoids placing password directly on the command line, which could show up in logs or process lists. Jenkins keeps the password hidden using its credentials system, and this line uses it just-in-time for the login, then clears it from memory after.



<ins>pushing  the image to Docker Hub 

       sh "docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}" 
pushes the docker image to the assigned docker repostitory as specified in the environment variable, tags the image as well, as its assigned in the environement variable


<ins>finally logging out of docker

        sh "docker logout"




Stage 4: Run Docker Container

            stage('Run Docker Container') {
                    steps {
                        script {
                            sh "docker run -d -p 8081:80 ${DOCKER_IMAGE}"


this stage starts the newly build container in detached mode (-d) and maps port 8081 on the host to port 80 on the container 

*Note: We have chosen port 8081 instead of 8080 because Jenkins is already using port 8080. If both Jenkins and Docker were assigned the same port, it would cause an error during the Docker build process.*

If Port 8081 is being used / running on a existing build. We can terminate the process by running a terminate process, then restarting docker 

    $ sudo lsof -i :<PID>
    $ sudo kill -9 <PID>

 ![alt text](<img/img 33..png>) 

*PID means process id*

![alt text](<img/img 34..png>)


### Configuring Build Trigger 
Like we did earlier in the freestyle project, We've selected our Build trigger as "Github hook trigger for GITScm polling" which allows builds to be triggered automatically when changes occur in the repository.

![alt text](<img/img 36..png>)

Once the script is fully configured, we can save, apply and trigger a build from our github repository. 

Pushing the files we have created earlier on our EC2 instance (`dockerfile` and `index.html`) will trigger Jenkins to automatically run a new build for our pipeline.

![alt text](<img/img 38..png>)

We can test out the  application, by inputting our EC2 instance IP address along with the docker port in a URL browser

    <EC2_instance IP:8081>
![alt text](<img/img 39..png>)




