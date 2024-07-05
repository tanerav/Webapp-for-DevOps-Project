## CI/CD Pipeline on AWS Using Jenkins, Maven, Docker

![Image](/Images/CICD.png)

# Configure Jenkins Server

## Step 1: Configure Jenkins Server
First, we need to launch an EC2 instance with a Keypair.
- **Instance Type:** t2.micro
- **AMI:** Amazon Linux-2
- **Security Group:**
  - 22, SSH
  - 8080, Custom TCP

Once our server is running, connect to your server via SSH using your keypair.

Next, we need to install Java, Jenkins, and Maven on our server.

First, switch to the root user and update the packages:
```sh
sudo su -
sudo yum update -y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
```
Then we need to install Java
```sh
sudo amazon-linux-extras install java-openjdk11 -y
```
After installing Java, we can now install Jenkins and start our Jenkins server
```sh
sudo yum install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

Connect to http://<your_server_public_DNS>:8080 from your browser. You can get ```initialAdminPassword``` for jenkins by running below command:
```sh
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
We need to install git in our Jenkins server, run below command:
```
sudo yum install git
```

Now we can run our first Jenkins job. Go to ```Dashboard``` -> ```New Item```
```
JobName: HelloWorldJob
Type: Free Style Project
Build Step: Execute shell
echo "Hello World!"
```

## Step 2: Integrate Git with Jenkins
We need to go to ``Manage Jenkins`` -> ``Manage Plugins``. Here we will install Github Integration and Maven Integration plugins.

Now we can run our Second job to check Github integration is working as expected. Create another FreeStyleJob as below:
```
SCM: Git
URL: https://github.com/tanerav/Webapp-for-DevOps-Project.git
Save -> Build Now
```
You can check the Workspace, if your project successfully downloadded from GitHub. In Jenkins server, ``/var/lib/jenkins/workspace`` you can see the jobs you have run so far.

## Step 3: Install and Configure Maven in Jenkins Server
We need to install MAVEN in Jenkins server.
```
cd /opt
sudo wget https://dlcdn.apache.org/maven/maven-3/3.8.6/binaries/apache-maven-3.8.6-bin.tar.gz 
tar -xvzf apache-maven-3.8.6-bin.tar.gz 
mv apache-maven-3.8.6-bin maven
```

Next configure ``M2_HOME`` and ``M2``(binary directory) environment variables and add them to the PATH so that we can run maven commands in any directory. You can search where is your JVM by using ``tfind / name java-11*``

Now you need to edit .bash_profile to add these variables to path and save
```
M2_HOME=/opt/maven
M2=/opt/maven/bin
JAVA_HOME=/usr/lib/jvm/java-11-openjdk-11.0.16.0.8-1.amzn2.0.1.x86_64

PATH=$PATH:$HOME/bin:$M2_HOME:$M2:$JAVA_HOME
export PATH
```
To apply the changes we have made to .bash_profile, either we can logout and log back in or run ``source .bash_profile`` command. It will upload the changes.

## Step 4: Integrate Maven with Jenkins
Now go to Manage Jenkins and Global Tool Configuration to add path for Java and Maven.

We can configure a Maven Project to build our Code, go to ``Dashboard`` -> ``NewItem``
```
Name = FirstMavenProject
Type: Maven Project
Root Pom: pom.xml
Goals and options: clean install
```
now we can go to /var/lib/jenkins/workspace/FirstMavenProject/webapp/target to see our build artifact webapp.war file.

# Integrating Docker in CI/CD Pipeline
## Step 1: Setup Docker Environment
First we will create an EC2 instance for Docker and name it as Docker-Host.
```
instanceType: t2.micro
AMI: Amazon Linux-2
Security Group: 
22, SSH
8080, Custom TCP
```
Login to Docker-Host server via SSH, switch to root user ``sudo su -`` and install docker
``yum install -y``
## Step 2: Create Tomcat container
Go to DockerHub, search for Tomcat official image. We can pick a specific tag of the image, but if we don't specify it will pull the image with latest tag. For this project we will use latest Tomcat image.
```docker pull tomcat```
Next we will run docker container from this image.
```docker run -d --name tomcat-container -p 8081:8080 tomcat```
To be able to reach this container from browser, we need to add port 8081 to our Security Group. We can add a range of port numbers 8081-9000 to Ingress.

Once we try to reach our container from browser it will give us 404 Not found error. This is a known issue after Tomcat version 9. We will follow below steps to resolve this issue.

First we need to go inside container by running below command:
```
docker exec -it tomcat-container /bin/bash
```
Once we are inside the container, we need to move files under ``webapps.dist/`` to ``webapps/``
```
mv webapps webapps2
mv webapps.dist webapps
exit
```
However, this solution is temporary. Whenever we stop our container and restart the same error will be appeared. To overcome this issue, we will create a Dockerfile and create our own Tomcat image.
## Step 3: Create Customized Dockerfile for Tomcat
We will create a Dockerfile which will fix the issue.
```
FROM tomcat
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps/
```
Lets create an image from this Docker file.
```
docker build -t tomcat-fixed .
```
Now we can create a container from this image
```
docker run -d --name tomcat-container-fixed -p 8085:8080 tomcat-fixed
```
We can check our Tomcat server in browser now.

## Step 4: Integrate Docker with Jenkins
We will create a new user/password called ``dockeradmin`` and add it to docker group
```
useradd dockeradmin
passwd dockeradmin
usermod -aG docker dockeradmin
Start docker service.
systemctl status docker
systemctl start docker
systemctl enable docker
systemctl status docker
```
We can check our new user in ``/etc/passwd`` and groups ``/etc/group``.
```
cat /etc/passwd
cat /etc/group
```
Next we will allow username/password authentication to login to our EC2 instance. By default, EC2s are only allowing connection with Keypair via SSH not username/password.
```
vim /etc/ssh/sshd_config
```
We need to uncomment/comment below lines in sshd_config file and save it
```
PasswordAuthentication yes
#PasswordAuthentication no
```
We need to restart sshd service
```
service sshd reload
```
Go to Jenkins , install Publish over SSH plugin. next go to ``Manage Jenkins`` -> ``Configure System`` Find ``Publish over SSH`` -> ``SSH Server``. Apply changes and Save
```
Name: dockerhost
Hostname: Private IP of Docker Host(since Jenkins and Docker host are in the same VPC, they would be able to communicate over same network)
Username: dockeradmin
click Advanced
Check use password based authentication
provide password
```
We will create a Jenkins job with below properties:
```
Name: BuildAndDeployOnContainer
Type: Maven Project
SCM: https://github.com/tanerav/Webapp-for-DevOps-Project.git
POLL SCM: * * * * *
Build Goals: clean install
Post build actions: Send build artifacts over ssh
SSH server: dockerhost
TransferSet: webapp/target/*.war
Remove prefix: webapp/target
Remote directory: /home/dockeradmin
```
Save and build, we can check under dockerhost server if webapp successfully send to /home/dockeradmin by using SSH.

## Step 5: Update Tomcat dockerfile to automate deployment process
Currently artifacts created through Jenkins are sent to /home/dockeradmin. We would like to change it to home directory of root user, and give ownership of this new directory to dockeradmin
```
sudo su - 
cd /opt
mkdir docker
chown -R dockeradmin:dockeradmin docker
```
We have our Dockerfile under ``/home/root``, we will move Dockerfile under docker directory and change ownership as well
```
mv Dockerfile /opt/docker
chown -R dockeradmin:dockeradmin docker
```
We will change our Dockerfile to copy the webapps.war file under ``webapps/`` in Tomcat container.
```
FROM tomcat:latest
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps/
COPY ./*.war /usr/local/tomcat/webapps
```
we build the image and create a container from the newly created image.
```
docker build -t tomcat:v1 .
docker run -d --name tomcatv1 -p 8086:8080 tomcat:v1
```
We can check our app from browser ``http://<public_ip_of_docker_host>:8086/webapp/``

Now we can configure our ``BuildAndDeployOnContainer`` job to deploy our application. We will add below lines to Exec Command part. We also need to change Remote directory path as ``//opt//docker``
````
cd /opt/docker;
docker build -t regapp:v1 .;
docker stop registerapp;
docker rm registerapp;
docker run -d --name registerapp -p 8089:8080 regapp:v1n
```