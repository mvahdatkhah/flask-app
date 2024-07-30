# Prerequisites
## 1. Jenkins Setup:

* Ensure Jenkins is installed and running on your server.
* Install the necessary Jenkins plugins: Git, Docker Pipeline, AWS Credentials, and SSH Agent.

## 2. AWS Setup:

* Ensure you have an EC2 instance running.
* Set up SSH access to the EC2 instance.
* Install Docker and Docker Compose on the EC2 instance.

## 3. GitHub Repository:

* Your repository should have the following files:
	* `Dockerfile`
	* `docker-compose.yaml`
	* Flask application files (e.g., `app.py`, `requirements.txt`)

# Step-by-Step Guide
## 1. Create Jenkins Pipeline Job
### 1. Open Jenkins Dashboard:

* Go to Jenkins dashboard and click on “New Item”.
* Enter the name of your job and select “Pipeline” as the job type. Click “OK”.

### 2. Configure Pipeline Job:

* In the job configuration page, scroll down to the “Pipeline” section.

### 3. Pipeline Definition:

* Set the Definition to “Pipeline script from SCM”.
* Select “Git” as the SCM and enter the repository URL.
* Provide credentials if the repository is private.
* Set the branch to the desired branch (e.g., `main`).

### 4. Pipeline Script:

* Create a `Jenkinsfile` in the root of your GitHub repository with the following content:

```t
pipeline {
    agent any
    
    environment {
        // AWS Credentials and Region
        AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        AWS_REGION = 'us-east-1'
        
        // SSH Credentials ID
        SSH_CREDENTIALS_ID = 'ec2-ssh'
        
        // EC2 Instance details
        EC2_USER = 'ec2-user'
        EC2_IP = 'your-ec2-instance-ip'
    }
    
    stages {
        stage('Clone Repository') {
            steps {
                git url: 'https://github.com/your-username/your-repo.git', branch: 'main'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    sh 'docker build -t flask-app .'
                }
            }
        }
        
        stage('Test Application') {
            steps {
                script {
                    // Assuming you have tests defined, e.g., pytest
                    sh 'pytest'
                }
            }
        }
        
        stage('Deploy to EC2') {
            steps {
                script {
                    // Copy docker-compose.yaml and other necessary files to EC2
                    sshagent(['SSH_CREDENTIALS_ID']) {
                        sh """
                        scp docker-compose.yaml ${EC2_USER}@${EC2_IP}:/home/${EC2_USER}/
                        """
                    }
                    
                    // Connect to EC2 and run docker-compose
                    sshagent(['SSH_CREDENTIALS_ID']) {
                        sh """
                        ssh ${EC2_USER}@${EC2_IP} 'docker-compose -f /home/${EC2_USER}/docker-compose.yaml up -d'
                        """
                    }
                }
            }
        }
    }
}
```

## 2. Configure Jenkins Credentials
* AWS Credentials:

	* Go to Jenkins dashboard.
	* Manage Jenkins > Manage Credentials > (select the correct store) > Add Credentials.
	* Add your AWS Access Key ID and Secret Access Key.

* SSH Credentials:

	* Add your SSH private key used to access the EC2 instance in Jenkins credentials.


## 3. Set Up Docker and Docker Compose on EC2
* Ensure Docker and Docker Compose are installed and running on your EC2 instance. You can use the following commands:	

```t
# Install Docker
sudo yum update -y
sudo amazon-linux-extras install docker
sudo service docker start
sudo usermod -a -G docker ec2-user

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

## 4. Set Up docker-compose.yaml
* Ensure your docker-compose.yaml file includes Redis and Flask app configuration.
`docker-compose.yaml`:

```t
services:
  web-fe:
    build: .
    deploy:
      replicas: 1
    command: python app.py
    ports:
      - target: 8080
        published: 5001
    networks:
      - counter-net
    volumes:
      - type: volume
        source: counter-vol
        target: /app
  redis:
    image: redis:alpine
    deploy:
      replicas: 1
    networks:
      counter-net:

networks:
  counter-net:
volumes:
  counter-vol:
```

## 5. Run the Pipeline
* Save the Jenkins job configuration and trigger the build.
* Monitor the console output for any errors and ensure the steps complete successfully.

