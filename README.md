This repo will discuss automating code deployment based on changes in the repository, using a **Django Notes App** and a **NodeJS Todo Application** as examples.
## Code Deployment
##### 1. Manual Deployment
Developers can manually run the code on the server.
##### 2. Automation (CI/CD Pipeline) 
Automation involves the following stages: 
1. **Build**: Compile and prepare the code for deployment. 
2. **Test**: Execute tests to ensure code quality. 
3. **Deploy**: Push the code changes to the server.

-----
## Problem Statement 
When a developer writes code to be deployed on AWS, the choice of tools and deployment process is crucial. The solution in this repo involves leveraging DevOps tools like GitHub, Docker, Docker Hub, Jenkins, and AWS cloud.

### Methodology
Here's my proposed methodology:
![[Proposed Methodology.png]]

Let's use Jenkins to create a pipeline using Groovy. There are three options: 
1. Create a *FreeStyle Item* 
2. Develop a *pipeline triggered by a script* 
3. Establish a *declarative pipeline using the pipeline from source control* (GitHub). This method checks for correct syntax usage.

----

To start with, let's create an AWS EC2 Instance. I used Ubuntu Server 22.04 LTS as an image and allow SSH traffic from anywhere to connect to this instance. Refer to the following 4 images for further guidance:

![[1.jpg]]

![[2.jpg]]

![[3.jpg]]

![[4.jpg]]

Now, connect to EC2 Instance via local terminal.
![[connect_to_instance.jpg]]

Use to following commands to install Jenkins.

First, update application packaging tool.
```bash
sudo apt update
```

Then install Java.
```bash
sudo apt install openjdk-11-jre
```

Check if it is installed or not
```bash
java -version
```

Now, use the following commands to install Jenkins and get it started as a service.
```bash
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```

Check is Jenkins is running.
```bash
systemctl status jenkins
```

Go to `EC2 Instance` -> `Security Group`. Then add an inbound rule allowing port 8080, on which Jenkins is running, to be accessible anywhere using the public IP.
![[inbound rule.jpg]]

Now, go to `<public ip of EC2 instance>:<Port on which Jenkins is running>`
In our case, the url of Jenkins is: http://44.204.28.208:8080/

Type the following command to get the initial password.
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Create the first admin user after the prompt. 
![[create admin.jpg]]

Create an item on Jenkins from the menu having the following specification. Select Pipeline and choose the relevant name for it.
![[itemname.jpg]]

After that, set the specification of the pipeline as shown:
![[specification.jpg]]
![[specification2.jpg]]

Now, create a dummy Groovy to check if pipeline is working or not.
![[testcode.jpg]]

----
## Problems Encountered
1. To push the image to Docker Hub, login via Shell. For that, we have to put both username and password. Password being a sensitive information can't be put in Groovy so I had to find a workaround.

Go to `Manage Jenkins` -> `Security` ->  `Configure Credential` and add in the credential in `Global Credentials`. 
![[credential1.jpg]]
![[credential2.jpg]]

To catch those environment variables in the code, use the below code snippet.
```groovy
steps {
	echo "Pushing the image to Docker Hub"
	withCredentials([usernamePassword(credentialsId:"dockerHub", passwordVariable:"dockerHubPass", usernameVariable:"dockerHubUser")]) {
		sh "docker tag my-notes-app ${env.dockerHubUser}/my-notes-app:latest"
		sh "docker login -u ${env.dockerHubUser} -p ${env.dockerHubPass}"
		sh "docker push ${env.dockerHubUser}/my-notes-app:latest"
	}
}
```

Assigning a unique ID to each credential from which later we can extract Username and Password.
![[extractusernameandpassword.jpg]]

2. While using the below command to start the deployment of the app, I encountered a second problem.
```groovy
steps { 
	echo "Deploying the container" sh "docker run -d -p 8000:8000
}
```

After rebuilding, the port 8000 shows its already being used. This creates a problem as the goal is to auto deploy the pushed changes. So, I came up with a workaround which is `docker-compose.yaml` file.
```yaml
version : "3.3"
services :
  web :
     image : priyansh2120/my-notes-app:latest
     ports :
         - "8000:8000"
```

By using the following command, the container can be started.
```sh
docker-compose up
```

If this can preceded by the following command, we can easily stop the existing running app, if it is running
```sh
docker-compose down && docker-compose up -d
```

The final JenkinsFile should look like this:
```groovy
pipeline {
    agent any
    
    stages {
        stage("Clone code") {
            steps {
                echo "Cloning the code"
                git url:"https://github.com/priyansh2120/django-notes-app-for-deployment.git", branch: "main"
            }
        }
        stage("build") {
            steps {
                echo "Building the image"
                sh "docker build -t my-notes-app ."
            }
        }
        stage("Push to Docker Hub") {
            steps {
                echo "Pushing the image to Docker Hub"
                withCredentials([usernamePassword(credentialsId:"dockerHub", passwordVariable:"dockerHubPass", usernameVariable:"dockerHubUser")]){
                    sh "docker tag my-notes-app ${env.dockerHubUser}/my-notes-app:latest"
                    sh "docker login -u ${env.dockerHubUser} -p ${env.dockerHubPass}"
                    sh "docker push ${env.dockerHubUser}/my-notes-app:latest"
                    
                }
                
                
            }
        }
        stage("Deploy") {
            steps {
                echo "Deploying the container"
                sh "docker-compose down && docker-compose up -d"
            }
        }
    }
}
```

3. The final problem encountered is, if changes need to be made in JenkinsFile itself, it has to be done in Jenkins. We can do it from GitHub itself by following the below option which can be accessed by going to configure the item.
![[jenkinongithub.jpg]]
![[jenkinongithub2.jpg]]

---
## Trigger Automatic Deployment
The last step is to trigger automatic deployment whenever developer pushes to the main branch which can be don through the concept of **Webhooks**.
![[webhooks.jpg]]

Add the webhook as follows:
![[webhook2.jpg]]

Check is webhooks are added or not
![[webhook3.jpg]]

Now, whenever the developer will push to the main branch, the deployment will be automatic.
![[deployementautomatic.jpg]]


