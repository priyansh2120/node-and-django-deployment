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
![Proposed Methodology](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/abbf625f-6f60-494f-b21f-65ab20acf4a4)


Let's use Jenkins to create a pipeline using Groovy. There are three options: 
1. Create a *FreeStyle Item* 
2. Develop a *pipeline triggered by a script* 
3. Establish a *declarative pipeline using the pipeline from source control* (GitHub). This method checks for correct syntax usage.

----

To start with, let's create an AWS EC2 Instance. I used Ubuntu Server 22.04 LTS as an image and allow SSH traffic from anywhere to connect to this instance. Refer to the following 4 images for further guidance:

![1](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/745b1138-2b58-4509-93be-e56003c25a73)


![2](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/1970f890-0d37-4eb7-abb3-58cd56a6aa4d)


![3](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/5df0fea0-9f3e-4418-b85a-f2628ae2c89a)


![4](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/8f7ce908-fc42-4764-a01b-b6456711e58b)

Now, connect to EC2 Instance via local terminal.
![connect_to_instance](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/79a6b6c5-1093-416b-8d0c-1e98b4bf5baf)


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
![inbound rule](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/3a6f05a1-71ea-4eb4-928f-2263b9c85c49)


Now, go to `<public ip of EC2 instance>:<Port on which Jenkins is running>`
In our case, the url of Jenkins is: http://44.204.28.208:8080/

Type the following command to get the initial password.
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Create the first admin user after the prompt. 
![create admin](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/17d66a9e-c729-40d4-8a42-44b3c8cea429)

Create an item on Jenkins from the menu having the following specification. Select Pipeline and choose the relevant name for it.
![itemname](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/684e6b6e-c157-4464-ae0c-d6af3b98423a)


After that, set the specification of the pipeline as shown:
![specification](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/4e33c34c-0b2f-48cd-a0a7-f11209f56688)

![specification2](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/8a5da738-be68-4a4b-a6e7-c272a983b902)


Now, create a dummy Groovy to check if pipeline is working or not.
![testcode](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/41d060f0-e65a-466f-8c69-2dd25d5f7c73)


----
## Problems Encountered
1. To push the image to Docker Hub, login via Shell. For that, we have to put both username and password. Password being a sensitive information can't be put in Groovy so I had to find a workaround.

Go to `Manage Jenkins` -> `Security` ->  `Configure Credential` and add in the credential in `Global Credentials`. 
![credential1](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/d9a4102e-cb8f-4526-89e8-3e8219fbe6b2)

![credential2](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/070e1a7f-35e6-4d66-95c1-2142bba69f88)


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
![extractusernameandpassword](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/6ef2b8bc-ba0d-4014-8d17-595a10eab0d9)


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
![jenkinongithub](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/c8beed4f-3481-41dc-b179-1166d9ba94f2)

![jenkinongithub2](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/587aa189-b0b7-4e79-871d-69cb99ae1384)

---
## Trigger Automatic Deployment
The last step is to trigger automatic deployment whenever developer pushes to the main branch which can be don through the concept of **Webhooks**.
![webhooks](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/ec0569d9-96f2-4dc4-b777-ca2777db5cd7)


Add the webhook as follows:

![webhook2](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/8c6b141f-05ed-433f-9585-786afe0eadb1)

Check is webhooks are added or not
![webhook3](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/15ccfced-0e51-4977-855a-fd94b6283c44)


Now, whenever the developer will push to the main branch, the deployment will be automatic.
![deployementautomatic](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/4c00e38a-44a4-406f-af4b-58844617bcf2)

The Deployed Notes App
![image](https://github.com/priyansh2120/node-and-django-deployment/assets/96059277/8c87a316-d5d0-403d-a212-71f921e7381c)

