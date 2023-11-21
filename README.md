Building a Docker-Jenkins CI/CD Pipeline for a Python App !!

Introduction:

In this article, we will look at how we can deploy an app using a CI/CD pipeline involving git, GitHub, Jenkins, Docker and DockerHub. The basic premise is that when a code update is pushed to git, it will get updated on GitHub. Jenkins will then pull this update, build the Docker Image from a Dockerfile and Jenkinsfile configuration, push it to Docker Hub as a registry store, and then pull it and run it as a container to deploy our app.

Prerequisites:

We will use a Python app for this tutorial. The sample app will be included in the GitHub repo.
GitHub account to sync our local repo and connect with Jenkins.
Docker Hub account. If you do not already have one, you can create it at hub.docker.com

Installing/Updating Java
First we will check if Java is installed and what version is it.
java -version
command : sudo apt-get install -y openjdk-11-jre

Installing Git:

Git will help us in maintaining and versioning our code in an efficient manner.

First let us check if Git is already available in our system or not.
command:git --version
you can install it using this command:
sudo apt-get install -y git

Configuring Git (Local Repo)
Let's first create a folder for our project. We will be working inside this folder throughout the tutorial.
mkdir pythonapp
We will initialize our local Git repository inside this folder.
cd pythonapp
But before we initialize our local repository, we need to make some changes to the default Git configuration.
git config --global init.defaultBranch main
By default, Git uses 'master' as the default branch. However, GitHub and most developers like to use 'main' as the default branch.

Further, we will also configure our name and email ID for Git.
git config --global user.name "your_name"
git config --global user.email "your@email.com"
To verify your modifications to the Git configuration, you can use this command:
git config --list

Now it's time to initialize our local repository.
git init
Initialising Git

This will create an empty repository in the folder. You can also alternatively create a repository on GitHub first and then clone it to your local system.

Setting up GitHub (Remote Repo)
Our local Git repository is not setup and initialized. We will now create a remote repo on GitHub to sync with local.

Login to your GitHub account and click on your Profile picture. Click on 'Your Repositories'.

On the page that opens, click on the green 'New' button.

Let's name our repo 'pythonapp' to keep it same as our folder name. This is not necessary but it will keep things simpler.

Creating GitHub Repository

Keep the repository as 'Public' and click on 'Create Repository'

Connecting to GitHub
For this tutorial, we will use SSH to connect the local repo to our remote repo. Please note that GitHub has stopped allowing username/password combinations for connections. If you wish to use https instead, you can check out this tutorial to connect using Personal Access Tokens.

First we will create an SSH key in our Ubuntu system.
ssh-keygen
Press 'enter' three times without typing anything.

Generating an SSH Keypair

This will create an SSH key in your system. We will use this key in our GitHub account. To access the key, use this command
cat ~/.ssh/id_rsa.pub
Copy the entire key.

On GitHub, go to your repository and click on 'Settings'.

On the left, in the 'Security' section, click on 'Deploy Keys'.

GitHub SSH Key Addition

Name the key to whatever you wish. Paste the key that you copied from the terminal inside the 'Key' box. Be sure to tick the 'Allow Write Access' box.

Now click on 'Add Key'. We now have access to push to our remote repo using SSH.

Now we will add the remote that will allow us to perform operations to the remote repo.
git remote add origin git@github.com:archanakaushish/pythonapp.git
To verify your remote
git remote
Verifying our Git Remotes

To verify and connect our configuration, we will do
ssh -T git@github.com
When prompted, type 'yes'. You should see a message that says 'You have successfullly authenticated, but GitHub does not provide shell access.'

GitHub SSH Connection Verifiction

Python App
Let's create Python app that will display Hello World! in the browser when executed.

Inside your terminal, make sure you are in the project folder. Create a folder named 'src' and create a file name 'helloworld.py' inside this folder like this:
mkdir src
cd src
sudo nano helloworld.py
Now let's write a Python script! Inside the nano editor, type this:
from flask import Flask, request
from flask_restful import Resource, Api

app = Flask(__name__)
api = Api(app)

class Greeting (Resource):
    def get(self):
        return 'Hello World!'

api.add_resource(Greeting, '/') # Route_1

if __name__ == '__main__':
    app.run('0.0.0.0','3333')

    CTRL + X to save .then enter

    Installing Jenkins
We now have the basics ready for deploying our app. Let's install the remaining software to complete our pipeline.

We begin by importing the GPG key which will verify the integrity of the package.
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
Next, we add the Jenkins softwarey repository to the sources list and provide the authentication key.
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
Jenkins Key and Source

Now, we install Jenkins
sudo apt-get install -y jenkins
Wait till the entire installation process is over and you get back control of the terminal.

Installing Jenkins

To verify if Jenkins was installed correctly, we will check if the Jenkins service is running.
sudo systemctl status jenkins.service
Verifying Jenkins Installation

Press Q to regain control.

Jenkins Configuration
We have verified that the Jenkins service is now running. This means we can go ahead and configure it using our browser.

Open your browser and type this in the address bar:
localhost:8080
You should see the Unlock Jenkins page.

Unlocking Jenkins for First Use

Jenkins generated a default password when we installed it. To locate this password we will use the command:
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
Jenkins first unlock password

Copy this password and paste it into the box on the welcome page.

On the next page, select 'Install Suggested plugins'

Jenkins Suggested Plugins

You should see Jenkins installing the plugins.

Jenkins Plugins Installation

Once the installation has completed, click on Continue.

On the Create Admin User page, click 'Skip and Continue as Admin'. You can alternatively create a separate Admin user, but be sure to add it to Docker group.

Click on 'Save and Continue'

On the Instance Configuration page, Jenkins will show the URL where it can be accessed. Leave it and click 'Save and Finish'

Click on 'Start Using Jenkins'. You will land on a welcome page like this:

Jenkins Landing Page

We have now successfully setup Jenkins. Let's go back to the terminal to install Docker.

Installing Docker
First we need to uninstall any previous Docker stuff, if any.
sudo apt-get remove docker docker-engine docker.io containerd runc
Most likely, nothing will be removed since we are working with a fresh install of Ubuntu.

We will use the command line to install Docker.
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
Docker Pre-requisites

Next, we will add Docker's GPG key, just like we did with Jenkins.
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
Now, we will setup the repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
Next we will install the Docker Engine.
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
Docker Installation

Now verify the installation by typing
docker version
Docker version

Notice that you will get an error for permission denied while connecting to Docker daemon socket. This is because it requires a root user. This means you would need to prefix sudo every time you want to run Docker commands. This is not ideal. We can fix this by making a docker group.
sudo groupadd docker
The docker group may already exist. Now let's add the user to this group.
sudo usermod -aG docker $USER
Apply changes to Unix groups by typing the following:
newgrp docker
Note: If you are following this tutorial on a VM, you may need to restart your instance for changes to take effect.

Let's verify that we can now connect to the Docker Engine.
command:docker version
Docker Engine Version

As we can see, Docker is now fully functional with a connection to the Docker Engine.

We will now create the Dockerfile that will build the Docker image.

Creating the Dockerfile
Inside your terminal, within your folder, create the Dockerfile using the nano editor.
command:sudo nano Dockerfile
Type this text inside the editor:

FROM python:3.8
WORKDIR /src
COPY . /src
RUN pip install flask
RUN pip install flask_restful
EXPOSE 3333
ENTRYPOINT ["python"]
CMD ["./src/helloworld.py"]
Building the Docker Image
From the Dockerfile, we will now build a Docker image.
docker build -t helloworldpython .
Building the Docker Image

Now let's create a test container and run it a browser to check if our app is displaying correctly.
command : docker run -p 3333:3333 helloworldpython

## Creating the Jenkinsfile

We will create a Jenkinsfile which will elaborate a step-by-step process of building the image from the Dockerfile, pushing it to the registry, pulling it back from the registry and running it as a container.

Every change pushed to the GitHub repository will trigger this chain of events.



```bash
sudo nano Jenkinsfile
In the nano editor, we will use the following code as our Jenkinsfile.
node {
    def application = "pythonapp"
    def dockerhubaccountid = "nyukeit"
    stage('Clone repository') {
        checkout scm
    }

    stage('Build image') {
        app = docker.build("${dockerhubaccountid}/${application}:${BUILD_NUMBER}")
    }

    stage('Push image') {
        withDockerRegistry([ credentialsId: "dockerHub", url: "" ]) {
        app.push()
        app.push("latest")
    }
    }

    stage('Deploy') {
        sh ("docker run -d -p 3333:3333 ${dockerhubaccountid}/${application}:${BUILD_NUMBER}")
    }

    stage('Remove old images') {
        // remove old docker images
        sh("docker rmi ${dockerhubaccountid}/${application}:latest -f")
   }
}
Explaining the Jenkinsfile
Our Jenkins pipeline is divided in 5 stages as you can see from the code.

Stage 1 - Clones our Github repo
Stage 2 - Builds our Docker image from the Docker File
Stage 3 - Pushes the image to Docker Hub
Stage 4 - Deploys the image as a container by pulling it from Docker Hub
Stage 5 - Removes the old image to stop image pile up.
Now that our Jenkinsfile is ready, let's push all of our source code to GitHub.

Pushing files to GitHub
First, let's check the status of our local repo.
command:git status
Git not tracking files

As we can see, there are no commits yet and there are untracked files and folders. Let's tell Git to track them so we can push them to our remote repo.
command:git add *
This will add all the files present in the git scope.

Git is now tracking our files and they are ready to be commit. The commit function pushes the files to the staging area where they will be ready to be pushed.
command:git commit -m "First push of the python app"
First commit to Git

Now, it's time to push our files.
command:git push -u origin main
Let's go to our repo on GitHub to verify that our push was successful.

Verifying the first push to GitHub




