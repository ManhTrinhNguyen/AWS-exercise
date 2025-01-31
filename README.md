

# "Build Automation with Jenkins"

##### Test
The project uses jest library for tests. (see "test" script in package.json)
There is 1 test (server.test.js) in the project that checks whether the main index.html file exists in the project. 

To run the nodejs test:

    npm run test

Make sure to download jest library before running test, otherwise jest command defined in package.json won't be found.

    npm install

In order to see failing test, remove index.html or rename it and run tests.

## Step 1 : Dockerize Node App 
    ```
        ## To run Node App i need Nodejs as a Base image
        FROM node:20-alpine 
        
        ## RUN (Execute any Linux cmd inside the container) . 
        ## I Create a /home/app dir inside the container
        RUN mkdir -p /home/app 
        
        ## COPY (To Copy from the App in the Host to Destination in the container)
        COPY ./app /home/app 
        
        ## WORKDIR : Make sure the container working on this Dir
        WORKDIR /home/app 
        
        ## Install npm 
        RUN npm install 
        
        ## CMD : Help me execute the entrypoint
        CMD [ "node", "server.js" ]

    ```

## Step 2 : Create a full pipeline for NodeJS App
1. Rent a EC2 sercer on AWS to run Jenkins as a Docker container 
    - **Step to create EC2 Instance :**
        ```
            1. Add Tags
            2. Choose OS Image
            3. Choose Capacity
            4. Network Configuration
            5. Configure Security Group
            6. Add storage
        ```

    - **Connect to Intance :**
        ```
            1. Move .pem (key pair) to more secure location ~./ssh/
            2. Change persion of .pem to make permission stricter with only me as a user can read that (chmod 400 instancefile)
            3. Connect to instance : ssh -i ~/.ssh/docker-server.pem ec2-user@Ipv4
                    -i: This flag will take the pem file as a parameter
                    ec2-user: I need to ssh to instance as a ec2 user not as a root user
        ```

    - **Install Docker :**
    ```
    - Update Package Manager: `sudo yum update`
    - Install Docker: `sudo yum install docker` 
    - When docker installed I have to start docker Daemon : sudo service docker start
    - I want to run docker command without using sudo . I will add user to Docker group: sudo usermod -aG docker $USER . After running  this command user may not add to docker group yet . I need to exit and login again
    ```

    - **Install Jenkins**
    ```
     - Run Jenkins as a Docker Container: `docker run -d -p 8080:8080 -p 50000:50000 -v jenkins-home:/var/jenkins-home -v /var/run/docker.sock:/var/run/docker.sock jenkins/jenkins`
    
        -d: Detach mode
        -p 8080:8080 : Jenkins run on port 8080
        -v jenkins-home:/var/jenkins-home: Create jenkins volumn to persist data
        -v /var/run/docker.sock:/var/run/docker.sock: This will mount docker CLI to Jenkins container so docker CLI will available in jenkins container
        jenkins/jenkins: Jenkins Image

    - Install Docker Inside Jenkins container so I have Docker CLI available in Jenkins : `curl https://get.docker.com/ > dockerinstall && chmod 777 dockerinstall && ./dockerinstall`
    - Inside jenkins container as root: `chmod o=rw /var/run/docker.sock` to make other user can use docker CLI in the container
    ```
    
2. When Jenkins available I can take the IP Address and the port to access. 
    ```
    - To create a Node Pipeline I need to configure Credentials to get access to source code 
    - Configure Nodejs so that Node will be available in Jenkins 
    - Configure Webhook Trigger pipeline to trigger pipeline whenever developer push code to a Repo
    - Create Jenkinsfile to create pipeline 
        - From Pipeline enable : GitHub hook trigger for GITScm polling
        - From Github : `In the Repo go to : Setting -> Webhook -> Add Webhook`
    ```

    - **Dynamic increment App version in Nodejs**
    ```
    1. npm install -D auto-version-js
    2. 
     - npx auto-version --patch # +0.0.1
     - npx auto-version --minor # +0.1.0
     - npx auto-version --major # +1.0.0
     - npx auto-version # no args is equivalent to --patch
    3. Install Plugin to use "readJSON file" : Pipeline: Utility Steps 
    ```

    - **Auto Commit to Git and**
    ```
    NOTE: Make sure to user withCredentials to get USER and PASS
    1. Connect to Git Repo Source : 
        - git remote set-url origin https://${USER}:${PASS}@github.com/ManhTrinhNguyen/jenkins-exercise.git
    2. Configure user name and email to set Stop the Webhook (When it auto commit and push Jenkins will not re run the pipeline job)
        - git config --global user.email jenkins@gmail.com
        - git config --global user.name Jenkins
    3. Add, Commit and Push
        - git add .
        - git commit -m "ci: bump version"
        - git push origin HEAD:main

    4 . Stop Web Hook to not re-run by using username :
        - In the Pipeline jobs -> Configure -> Additional Behavior -> Polling ignores commits from certain users
    ```

## Step 3 : : Extract into Jenkins Shared Library 

    1. **Create My shared lib Repo** (https://github.com/ManhTrinhNguyen/Jenkins-Docker-Excercise-Shared-Library)
    2. Get My Share Lib :
    ```
        library identifier: 'jenkins-shared-library@main', retriever: modernSCM(
            [$class: 'GitSCMSource',
             remote: https://github.com/ManhTrinhNguyen/Jenkins-Docker-Excercise-Shared-Library.git,
             credentialsId: 'github-credentials'
            ]
        )

    3. Or I want to put my Shared Lib in Global : 
        - From Jenkins Manage -> Systems -> Global Trusted Pipeline Libraries -> Configure name, github url, credentials 
        - In Jenkinsfile : @Library('name of my library shared')
    ```

# AWS Service 

## Create IAM User 
  - I using AWS UI to create User and Group 
  - Give Group the EC2fullAccess permission 

## Configure AWS CLI 
  ```
    - Used : aws configure 
      - Set credentials for that User so User can connect with AWS CLI 
      - Set Region for AWS CLI 
  ```

## Create VPC 
  ```
    I use AWS CLI to create VPC 
      - Create VPC with specific cidr block : aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query Vpc.VpcId --output text
        Will return vpc-id 

      - Create Subnet with VPC id : aws ec2 create-subnet --vpc-id vpc-0f8d575ef969edce8 --availability-zone us-west-1a --cidr-block 10.0.224.0/20 --query Subnet.SubnetId --output text
        Will return subnet-id
    
    I use these tool to calulate and divide IP range:
      IP Calculator: https://mxtoolbox.com/subnetcalculator.aspx
      IP Calculator with binary values: http://jodies.de/ipcalc
      Calculate sub-CIDR blocks: http://www.davidc.net/sites/default/subnets/subnets.html 
  ```

## Create and Attach an Internet Gateway
  ```
    - Create The internet Gateway (Router): aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text
    igw-08389efafdfeadd2f

    - Attach the IGW (Router) to a VPC (Virtual Private Cloud): aws ec2 attach-internet-gateway --internet-gateway-id igw-08389efafdfeadd2f --vpc-id vpc-0d434a4d2b1901c4b

    Then add the internet route (0.0.0.0/0) to IGW : aws ec2 create-route --route-table-id rtb-044ee3835233a7547 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-0d9eb190b5308b22b
  ```

## Create Security Group 
  ```
    - Create Security Group: aws ec2 create-security-group --group-name my-sg --description "My SG" --vpc-id vpc-xxxx
    - Create Security Groups rules: aws ec2 authorize-security-group-ingress --group-id sg-xxxx --protocol tcp --port 22 --cidr 198.27.191.24/32
    Create-keypair: aws ec2 create-key-pair --key-name MyKpCli --query 'KeyMaterial' --output text > MyKpCli.pem`
  ```

## Create EC2 Instance 
  ```
    - Create ec2 instance : aws ec2 run-instances --image-id ami-0fca1aacaa1ed9168 --count 1 --instance-type t2.micro --key-name tim --security-group-ids sg-0dcc26b2309569b5b --subnet-id subnet-0d220d84c63cad99e
  ```

## Install Docker on EC2 
  ```
    - SSH to EC2 : ssh -i key.pem ec2-user@ip-address-public
    - Update Package Manager : sudo yum update
    - Install Docker : sudo yum install docker
    - Start Docker : sudo service docker start
    - Make Docker available for other user with using Sudo : usermod -aG docker ${USER} . After this I need to exit and login again
  ```

## Connect EC2 server to Jenkins server vi ssh (ssh agent)
  ```
    - Install ssh agent plugin from Jenkins : Go to Manage -> Plugin -> SSH agent . This plugin allow me to use SSH credentials of the EC2 server to SSH into my instance
    - Create EC2 Credentials in Global Scope : Choose SSH username with private key -> paste my .pem key to that
    - Open ssh port allow Jenkins to access : Configure firewall
    - Connect EC2 and Run Docker in Jenkinsfile : 
  ```

## Using Docker Compose for Deployment 
  ```
    - Install docker-compose on EC2 instance : sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose . Docker-compose not in the Yum package so I use this command to install. - - Then make it executable sudo chmod +x /usr/local/bin/docker-compose
    - Create Docker-compose yaml file
    -  Adjust Jenkins to execute docker-compose command on EC2 instance 1. In the ssh agen block: copy docker-compose to EC2 sh "scp docker-compose.yaml ec2-user@54.215.92.78:/home/ec2-user"
  ```