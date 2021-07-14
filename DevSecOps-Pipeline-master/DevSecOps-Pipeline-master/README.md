# DevSecOps Pipline

*This walkthrough is based on Ubuntu 18.04, you may wish to tweak certain steps based on the OS you are using.

## Content Page

  1. **Setting up Jenkins Server**
        1. SSH
        2. Install Java
        3. Install Jenkins
        4. Activate Jenkins from  /var/lib/Jenkins/secrets/initialAdminPassword
        5. Install Plugins
        6. Install Docker
        7. Install Maven
  2. **Setting up Tomcat Server (Optional)**
       1. SSH
       2. Install Java
       3. Download and unzip Tomcat: unzip + wget
       4. Create Tomcat User and Allow Manager access from another hosts by commenting access restrictions in /webapps/manager/META-INF/context.xml
        3. **Creating Build Pipeline in Jenkins**
        5. Create New Item (Pipeline)
            3. Connect to GitHub repo
        6. Create Jenkinsfile to include Stages
        7. Run Pipeline
  4. **Integrating Continuous Deployment in Jenkins**
       1. Install Deploy to Container, SSH Agent Plugin
       2. Create Credentials for SSH to Prod Server
       3. Add 'Deploy-To-Tomcat Stage'
  5. **Checking Git Secrets in Pipeline using Trufflehog**
       1. Trufflehog - Regex based Scanner for git secrets
       2. sh 'docker pull gesellix/trufflehog'
       3. sh 'docker run -t gesellix/trufflehog --jason `<insert-github-repo-link-here>` > trufflehog'
  6. **Source Composition Analysis in Pipeline using OWASP Dependency Checker**
       1. Download owasp-dependency-check.sh to Jenkins workspace
       2. Create Source-Composition-Analysis stage and run the script
       3. sudo usermod -a -G docker jenkins
       4. systemctl restart jenkins
  7. **Sonarqube integration in Jenkins Pipeline**
       1. Run Sonarqube in container `500mb extra ram requirement`
       2. Default credentials - admin:admin
       3. Generate sonarqube Token: My Account -> Security - Generate Token
       4. Install Sonarqube plugin and Configure Systems in Jenkins
       5. Configure Sonarqube in Global Tool Configuration and select automatic installation
  8. **ZAP Baseline Scan Integration in Jenkins Pipeline**
       1. Memory Issues faced running ZAP on EC2 Jenkins Host `heavy ram requirement`
       2. Created dedicated Server for ZAP
       3. Configure SSH Credentials in Jenkins
       4. Add DAST step in pipeline
       5. Allow on failure --- `cmd || true`





## Setting Up Jenkins Server ##

> Pre-requisite for this step: You have set up two separate EC2 instances with proper configurations including Security Group allowing port:8080 (for jenkins) and port:9000 (for sonarqube)
>
> From this point forward, we will be using EC2 Ubuntu 18.04 instances (2 vCPUs + 4 Gib Memory)

### SSH ###

1. Enter your 1st EC2 (for jenkins) instance's DNS into PuTTY

   > PuTTY -> Session -> Host Name (or IP address)

2. Attach Private key file (that should have been downloaded when setting up EC2 instance) for authentication

   > PuTTY -> SSH -> Auth -> Browse Private key file for authentication (.ppk) -> Open

   If your private key file is .pem, then use PuTTY Key Generator to generate a .ppk file

   > PuTTY Key Generator -> Conversions -> Import Key (find your .pem file) -> Save Private Key 

3. Automate login username: ubuntu

   > PuTTY -> Connection -> Data -> Auto-login username -> Enter <ubuntu>

4. Save session settings

   > PuTTY -> Session -> Saved Sessions -> Enter <session-name> -> Save

5. Open PuTTY session

   > PuTTY -> Session -> Saved Sessions -> Load -> Open



### Install Java ###

1. First update the package index

   ```shell
   sudo apt-get update
   ```

   

2. Next install Java. This command will install the Java Runtime Environment (JRE)

   ```shell
   sudo apt-get install default-jre
   ```

   

3. Install Java JDK

   ```shell
   sudo apt-get install default-jdk
   ```

   

4. Install the Oracle JDK

   ```bash
   sudo apt install apt-transport-https && sudo apt update && sudo apt install -y openjdk-8-jdk
   ```

   


Reference: [Install Java with Apt-get]: https://www.digitalocean.com/community/tutorials/how-to-install-java-with-apt-get-on-ubuntu-16-04



### Install Jenkins ###

1. Install Jenkins

   ```bash
   wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
   #
   sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \
       /etc/apt/sources.list.d/jenkins.list'
   #
   sudo apt-get update
   #
   sudo apt-get install jenkins
   ```

2. Register Jenkins service

   ```bash
   sudo systemctl daemon-reload
   ```

3. Start Jenkins

   ```bash
   sudo systemctl start jenkins
   ```

4. Check status of Jenkins service

   ```
   sudo systemctl status jenkins
   ```

5. If everything has been set up correctly, you should see an output like this:

   ```bash
   ● jenkins.service - LSB: Jenkins Automation Server
      Loaded: loaded (/etc/rc.d/init.d/jenkins; generated)
      Active: active (running) since Thu 2021-07-08 06:30:44 UTC; 3min 4s ago
   ```

6. Access Jenkins Console on browser

   > http://jenkins:8080

   Replace `jenkins` with your EC2 public IP/DNS

Reference: [Install Jenkins on Ubuntu]: https://www.jenkins.io/doc/book/installing/linux/

### Activate Jenkins from  /var/lib/Jenkins/secrets/initialAdminPassword ###

1. Once connected to Jenkins Console, you will be prompted to `Unlock Jenkins` , go back to your SSH:

   ```bash
   sudo cat /var/lib/Jenkins/secrets/initialAdminPassword
   ```

2. Copy the password and paste it in Jenkins Console (fyi highlighting text in PuTTY will automatically *copy* it to clipboard and *paste* text by right-clicking in PuTTY )



### Install Plugins ###

1. Jenkins will now prompt "Install suggested plugins" or "Select plugins to Install"

   > Select "Install suggested plugins"

2. Install other plugins

   > Manage Jenkins -> Manage Plugins -> Available -> Search and select:
   >
   > 1) Maven Integration
   >
   > 2) SSH Agent
   >
   > 3) Deploy to container
   >
   > 4) Blue Ocean
   >
   > -> Install without restart
   
3. Enable Git

   > Manage Jenkins -> Global Tool Configuration -> Git
   >
   > > Install automatically: Checked

### Install Docker ###

1. First, update your existing list of packages:

```bash
sudo apt update
```

2. Next, install a few prerequisite packages which let `apt` use packages over HTTPS:

```bash
sudo apt install apt-transport-https ca-certificates curl software-properties-common
```

3. Then add the GPG key for the official Docker repository to your system:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

4. Add the Docker repository to APT sources:

```bash
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
```

5. Next, update the package database with the Docker packages from the newly added repo:

```bash
sudo apt update
```

6. Make sure you are about to install from the Docker repo instead of the default Ubuntu repo:

```bash
apt-cache policy docker-ce
```

7. You’ll see output like this, although the version number for Docker may be different:

   Output of apt-cache policy docker-ce

   ```bash
   docker-ce:
     Installed: (none)
     Candidate: 18.03.1~ce~3-0~ubuntu
     Version table:
        18.03.1~ce~3-0~ubuntu 500
           500 https://download.docker.com/linux/ubuntu bionic/stable amd64 Packages
   ```

   Notice that `docker-ce` is not installed, but the candidate for installation is from the Docker repository for Ubuntu 18.04 (`bionic`).

8. Finally, install Docker:

   ```shell
   sudo apt install docker-ce
   ```

9. Docker should now be installed, the daemon started, and the process enabled to start on boot. Check that it’s running:

   ```bash
   sudo systemctl status docker
   ```

   The output should be similar to the following, showing that the service is active and running:

   ```bash
   Output● docker.service - Docker Application Container Engine
      Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
      Active: active (running) since Thu 2018-07-05 15:08:39 UTC; 2min 55s ago
        Docs: https://docs.docker.com
    Main PID: 10096 (dockerd)
       Tasks: 16
      CGroup: /system.slice/docker.service
              ├─10096 /usr/bin/dockerd -H fd://
              └─10113 docker-containerd --config /var/run/docker/containerd/containerd.toml
   ```

   Installing Docker now gives you not just the Docker service (daemon) but also the `docker` command line utility, or the Docker client.

10. If you want to avoid typing `sudo` whenever you run the `docker` command, add your username to the `docker` group:

    ```bash
    sudo usermod -aG docker ${USER}
    ```

    Reference: [Install Docker on Ubuntu 18.04]: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-18-04

### Install Maven

1. Start by updating the package index:

   ```shell
   sudo apt update
   ```

2. Next, install Maven by typing the following command:

   ```shell
   sudo apt install maven
   ```

3. Verify the installation by running the `mvn -version` command:

   ```shell
   mvn -version
   ```

   The output should look something like this:

   ```output
   Apache Maven 3.5.2
   Maven home: /usr/share/maven
   Java version: 10.0.2, vendor: Oracle Corporation
   Java home: /usr/lib/jvm/java-8-oracle/jre
   Default locale: en_US, platform encoding: ISO-8859-1
   OS name: "linux", version: "4.15.0-36-aws", arch: "amd64", family: "unix"Copy
   ```

   That’s it. Maven is now installed on your system, and you can start using it.



Reference: [Install Maven on Ubuntu]:https://linuxize.com/post/how-to-install-apache-maven-on-ubuntu-18-04/

4. Add Maven Installation directory to Jenkins

   > Jenkins -> Manage Jenkins -> Global Tool Configuration -> Maven -> Maven Installations -> Add Maven 
   >
   > > Name: Maven
   > >
   > > Install automatically: **Unchecked**
   > >
   > > MAVEN_HOME: /usr/share/maven (error message will show if path is incorrect)
   >
   > -> Save 





## Setting up Tomcat Server ##

### SSH ###

Follow the guide above to open a SSH session with the other EC2 server that we are going to set up Tomcat on

### Install Java ###

Refer to guide above.

Reference: [Install Java with Apt-get]: https://www.digitalocean.com/community/tutorials/how-to-install-java-with-apt-get-on-ubuntu-16-04

### Download and unzip Tomcat ###

1. Copy this [link](https://downloads.apache.org/tomcat/tomcat-8/v8.5.69/bin/apache-tomcat-8.5.69.zip)

2. ```bash
   mkdir /prod
   cd /prod
   wget <paste link here>
   ```

3. Unzip the Tomcat file

   ```shell
   apt install unzip
   unzip apache-tomcat-8.5.39.zip
   ```

4. Start Tomcat

   ```
   cd apache-tomcat-.8.5.39.zip
   cd bin
   chmod +x catalina.sh
   bash startup.sh
   ```

5. Check if Apache Tomcat is running

   > http://tomcat:8080

   Replace `tomcat` with the corresponding EC2 public IP/DNS

6. Access Apache Tomcat Manager console

   > http://tomcat:8080/manager

   Most likely you will be thrown a ***403 Access Denied***

7. Modify `context.xml` file

   ```
   cd /prod/apache-tomcat-8.5.39/webapps/manager/META-INF
   vim context.xml
   ```

   > Comment out this part of the file
   >
   > >  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
   > >
   > > ​			allow="127\.\d+\\.\d+\\.\d+|::1|0:0:0:0:0:0:0:1" />

8. Edit `tomcat-users.xml`

   ```
   cd /prod/apache-tomcat-8.5.39/conf
   vim tomcat-users.xml
   ```

   > At the end of the file, insert these lines of code before `</tomcat-users>`
   >
   > > <role rolename=:manager-gui"/>
   > >
   > > <role rolename=:manager-script"/>
   > >
   > > <user username="tomcat" password="tomcat" roles"manager-gui,manager-script"/>
   >
   > Feel free to change the user name and password to your desired ones

9. Restart Tomcat

   ```
   cd /prod/apache-tomcat-8.5.39/bin
   bash shutdown.sh
   
   bash startup.sh
   ```

   > You should be able to see `Tomcat Web Application Manager` on http://tomcat:8080/manager





## Creating Build Pipeline in Jenkins ##

### Create New Item (Pipeline) ###

1. >  **Jenkins** -> **New Item** -> **Pipeline**

   #### General ####

   Enter name and description of pipeline
   
   > Discard old builds: Checked
   >
   > >  Max # of builds to keep: 2
   >
   > GitHub Project: Checked
   >
   > >Project Url: `Enter your github repo`
   
   #### Build Triggers ####
   
   > GitHub Hook trigger for GITScm Polling: Checked
   >
   > Poll SCM: Checked
   >
   > > Schedule: * * * * *
   > >
   > > This enables Jenkins to Poll GitHub for changes and new commits once every minute
   > >
   > > (Note: the space between each asterisk)
   
   #### Pipeline ####
   
   > Definition: Pipeline script from SCM
   >
   > > SCM: Git
   > >
   > > > Repository URL: `Enter your github repo`
   > > >
   > > > Credentials: None (Unless your github repo is private)
   > > >
   > > > Branches to build: */master
   > >
   > > Script Path: Jenkinsfile
   > >
   > > Lightweight checkout: checked
   
   and save.

### Create Jenkinsfile to include Stages ###

1. In your GitHub repo, create a new file called Jenkinsfile

2. For a simple pipeline, enter the following:

   ```
   pipeline {
     agent any 
     tools {
       maven 'Maven'
     }
     stages {
       
       stage ('Initialize') {
         steps {
           sh '''
                       echo "PATH = ${PATH}"
                       echo "M2_HOME = ${M2_HOME}"
               ''' 
         }
       }
       
       stage ('Build') {
         steps {
         sh 'mvn clean package'
          }
       }
   }
   ```

   ### Run Pipeline ###

   1. Now click `Build Now` to run pipeline
   2. `Open Blue Ocean` for more information

   

##  Integrating Continuous Deployment in Jenkins ##

### Store Tomcat SSH credentials in Jenkins ###

1. > Jenkins -> Credentials -> Jenkins -> Global Credentials (Unrestricted) -> Add some credentials
   >
   > > Kind: SSH Username with Private Key
   > >
   > > > Scope: Global (Jenkins, nodes, items, all child items, etc)
   > > >
   > > > ID: tomcat
   > > >
   > > > Description: tomcat SSH Credentials
   > > >
   > > > Username: ubuntu
   > > >
   > > > Private Key: Enter directly
   > > >
   > > > > Key: -----BEGIN RSA PRIVATE KEY-----
   > > > >
   > > > > ​		XXXXXXXX
   > > > >
   > > > > ​		-----END RSA PRIVATE KEY-----
   > > > >
   > > > > *(Note: Either copy and paste everything from your private key .pem file directly or use PuTTY Key Generator to convert .ppk file to .pem file and copy from there)*



### Add 'Deploy-To-Tomcat Stage' ###

1. Open your `Jenkinsfile` and insert the following:

   ```
   stage ('Deploy-To-Tomcat') {
               steps {
              sshagent(['tomcat']) {   
                sh 'scp -o StrictHostKeyChecking=no target/*.war ubuntu@18.117.153.231:/prod/apache-tomcat-8.5.68/webapps/webapp.war'
              
                 }      
              }       
       }
   ```

   Replace `ubuntu@18.117.153.231` with your own instance's IP

2. Change permission for `/prod/apache-tomcat-8.5.68/webapps/webapp.war`

   ```
   cd /prod/apache-tomcat-8.5.68/
   chmod 777 webapps/
   ```

3. Build Pipeline

   > If 'Deploy-to-Tomcat' stage failed and you see this error message:
   >
   > `scp: /prod/apache-tomcat-8.5.39/webapps/webapp.war: Permission denied`
   >
   > Changing the ownership for webapps in the server should solve this error
   >
   > `chown -R ubuntu:ubuntu  /prod/apache-tomcat-8.5.39/webapps`

4. Go to http://tomcat:8080/webapp/ to see the website successfully deployed.



## Checking Git Secrets in Pipeline using Trufflehog ##

### Trufflehog - *Regex based Scanner for git secrets* ###

Check out [Trufflehog](https://trufflesecurity.com/trufflehog) here

### Installing Trufflehog ###

> Via your Jenkins EC2 instance

1. ```bash
   docker pull gesellix/trufflehog
   ```

2. To check if your Trufflehog image is running

   ```bash
   docker images
   ```

3. Test run Trufflehog on your github repo

   ```bash
   docker run gesellix/trufflehog https://github.com/lumostay/DevSecOps-Pipeline.git
   ```

   Note: Feel free to replace with your own repo

4. Insert 'Check Git-Secrets' stage into `Jenkinsfile` by adding the following

   ```
   stage ('Check-Git-Secrets') {
         steps {
           sh 'rm trufflehog || true'
           sh 'docker run gesellix/trufflehog --json https://github.com/lumostay/DevSecOps-Pipeline.git > trufflehog'
           sh 'cat trufflehog'
         }
       } 
   ```

5. Build the pipeline again and see that there is a new stage 'Check-Git-Secrets'

   > If you face an error like this:  
   >
   > `docker: Got permission denied while trying to connect to the Docker daemon socket at unix...`
   >
   > Either you have not added the user into the Docker group or you have yet to restart the Jenkins Server within your instance after doing so. 
   >
   > 1. `sudo usermod -a -G docker jenkins`
   > 2. `systemctl restart jenkins`



## Source Composition Analysis in Pipeline using OWASP Dependency Checker ##

### Download owasp-dependency-check.sh to Jenkins workspace ###

Create a new file called `owasp-dependency-check.sh` and populate it with the code found in this [link](https://hub.docker.com/r/owasp/dependency-check).

### Create Composition Analysis in Pipeline using OWASP Dependency Checker ###

Insert the following 'Source Composition Analysis' stage after 'Check Git-Secrets' stage in `Jenkinsfile`:

```
stage ('Source Composition Analysis') {
      steps {
         sh 'rm owasp* || true'
         sh 'wget "https://raw.githubusercontent.com/lumostay/DevSecOps-Pipeline/master/owasp-dependency-check.sh" '
         sh 'chmod +x owasp-dependency-check.sh'
         sh 'bash owasp-dependency-check.sh'
         sh 'cat /var/lib/jenkins/OWASP-Dependency-Check/reports/dependency-check-report.xml'
        
      }
    }   
```

Build pipeline to check if stage has been added successfully 

## SAST: SonarQube integration in Jenkins Pipeline

### Installing SonarQube on instance ###

> If your Jenkins instance do not have enough memory, you can do this step on a separate instance

```shell
docker run -d -p 9000:9000 sonarqube
```

Check that SonarQube is running on http://Jenkins:9000

> Again, replace `Jenkins` with your Jenkins EC2 Public IP or DNS

Log in to SonarQube

> Default User: admin
>
> Deafult Password: admin

Generate Server authentication token

> SonarQube -> My Account (click top right user-profile icon)  -> Security -> Generate Tokens
>
> Take note of the generated token for the following steps

### Add Sonarqube plugin to Jenkins ###

> Jenkins -> Manage Jenkins -> Manage Plugins -> Available -> Search: `SonarQube Scanner` -> Install without restart

### Update  SonarQube credentials ###

> Jenkins -> Manage Jenkins -> Configure System -> SonarQube Servers

> Enable injection of SonarQube server config as build env var: Checked
>
> Name: sonar
>
> Server URL: http://Jenkins:9000 (Replace `Jenkins` with your instance public IP)
>
> Server authentication: <Paste your token here>

### Configure SonarQube in Global Tools ###

> Jenkins -> Manage Jenkins -> Global Tool Configuration -> SonarQube Scanner -> Add SonarQube Scanner

> Name: sonar
>
> Install automatically: Checked

### Add 'SAST' stage into Jenkinsfile ###

Add the following into your Jenkinsfile

```
stage ('SAST') {
      steps {
        withSonarQubeEnv('sonar') {
          sh 'mvn sonar:sonar'
          sh 'cat target/sonar/report-task.txt'
        }
      }
    } 
```

### Build Pipeline ###

Once the SAST stage passed, you can view the Analysis Report, where the link will be given in the console output, which looks something like http://13.233.252.1:9000/dashboard?id=lu.Webapp



## DAST: ZAP Baseline Scan Integration in Jenkins Pipeline ##

It is highly recommended to use a separate server dedicated for ZAP due to its heavy memory requirements.

Refer to the guide above to install Docker into the new server.

### Add ZAP credentials to Jenkins ###

> Jenkins -> Credentials -> Stores scoped to Jenkins -> Jenkins -> Global Credentials (unrestricted) -> Add Credentials 
>
> > Kind: SSH Username with private key
> >
> > Scope: Global (Jenkins, nodes, items, all child items, etc)
> >
> > ID: zap
> >
> > Description: zap ssh creds
> >
> > Username: ubuntu
> >
> > Private key: `Enter Private Key here` If you do not know where to find it, refer to guide above
> >
> > Passphrase: <Empty>



### Add 'DAST' Stage to Jenkins Pipeline ###

Add the following to your Jenkinsfile, as the last stage after 'Deploy-to-Tomcat' :

```
stage ('DAST') {
      steps {
        sshagent(['zap']) {
         sh 'ssh -o  StrictHostKeyChecking=no ubuntu@ZAP-IP-HERE "docker run -t owasp/zap2docker-stable zap-baseline.py -t http://Jenkins:8080/webapp/" || true'
        }
      }
    }
```

> Replace `ZAP-IP-HERE` and `Jenkins` with their respective IP addresses



### Build Pipeline ###

Once your entire pipeline has been executed successfully, you can check via Blue-Ocean, on the results of ZAP.

At the end of the console output, you should see something like:

```
FAIL-NEW: 0		FAIL-INPROG: 0 WARN-NEW: 11 	WARN-INRPOG: 0 INFO: 0 IGNORE: 0	PASS:13
```







# Congratulations!  #

# You have successfully created a DevSecOps Pipeline on Jenkins! #

