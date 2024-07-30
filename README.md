# flask-app
To create a Jenkins pipeline for deploying a simple Python and Flask application hosted on GitHub to an EC2 instance on AWS, follow the steps below. This pipeline will clone the repository, set up a virtual environment, install dependencies, run tests, and deploy the application to the EC2 instance.

Prerequisites:
1. Jenkins Configuration: Ensure Jenkins is set up with the necessary plugins (e.g., Git, Pipeline, SSH Agent, AWS CLI).
2. AWS Configuration: Ensure you have an EC2 instance running with SSH access configured.
3. Credentials: Ensure Jenkins has the necessary credentials for accessing GitHub and the EC2 instance.

## Jenkinsfile

pipeline {
    agent any

    environment {
        VIRTUAL_ENV = '.venv'
        FLASK_APP = 'app.py'
        EC2_USER = 'ec2-user' // Change to your EC2 username
        EC2_HOST = 'your-ec2-instance-public-ip' // Change to your EC2 public IP
        SSH_CREDENTIALS_ID = 'your-ssh-credentials-id' // Jenkins SSH credentials ID
    }

    stages {
        stage('Clone Repository') {
            steps {
                // Clone the GitHub repository
                git 'https://github.com/mvahdatkhah/flask-app.git'
            }
        }

        stage('Setup Python Environment') {
            steps {
                // Setup Python virtual environment
                sh 'python3 -m venv ${VIRTUAL_ENV}'
                sh '. ${VIRTUAL_ENV}/bin/activate'
            }
        }

        stage('Install Dependencies') {
            steps {
                // Activate virtual environment and install dependencies
                sh '. ${VIRTUAL_ENV}/bin/activate && pip install -r requirements.txt'
            }
        }

        stage('Run Tests') {
            steps {
                // Run your test suite
                sh '. ${VIRTUAL_ENV}/bin/activate && pytest'
            }
        }

        stage('Deploy to EC2') {
            steps {
                // Copy application files to the EC2 instance
                sshagent(credentials: [SSH_CREDENTIALS_ID]) {
                    sh '''
                    scp -o StrictHostKeyChecking=no -r * ${EC2_USER}@${EC2_HOST}:~/flask-app/
                    '''
                }
                // Connect to EC2 instance and run deployment script
                sshagent(credentials: [SSH_CREDENTIALS_ID]) {
                    sh '''
                    ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} << EOF
                    cd ~/flask-app
                    python3 -m venv ${VIRTUAL_ENV}
                    . ${VIRTUAL_ENV}/bin/activate
                    pip install -r requirements.txt
                    nohup flask run --host=0.0.0.0 > flask.log 2>&1 &
                    EOF
                    '''
                }
            }
        }
    }

    post {
        always {
            // Clean up virtual environment
            sh 'rm -rf ${VIRTUAL_ENV}'
        }

        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed.'
        }
    }
}

# Explanation:
1. Clone Repository: This stage clones your GitHub repository to the Jenkins workspace.
2. Setup Python Environment: This stage sets up a Python virtual environment.
3. Install Dependencies: This stage activates the virtual environment and installs the required Python packages from requirements.txt.
4. Run Tests: This stage runs the test suite using pytest.
5. Deploy to EC2: This stage performs the following steps:
	Copies the application files to the EC2 instance using scp.
	Connects to the EC2 instance and runs a deployment script to set up the virtual environment, install dependencies, and start the Flask application.

## Important Notes:
. Replace `https://github.com/yourusername/flask-app.git` with your actual GitHub repository URL.
. Replace `your-ec2-instance-public-ip` with your EC2 instance's public IP address.
. Replace `your-ssh-credentials-id` with the ID of your SSH credentials stored in Jenkins.
. Ensure your EC2 instance's security group allows incoming traffic on the port your Flask app will use (default is 5000).

# Additional Tips:
. Make sure Jenkins has access to Python and pip.
. If you're using any specific Jenkins plugins for Python, make sure they are installed and properly configured.
. Secure sensitive data like credentials using Jenkins credentials management rather than hardcoding them in the script.
. Consider using a process manager like gunicorn or supervisord for running your Flask application in a production environment.

By following this script, you should be able to set up a CI/CD pipeline for deploying your Python and Flask application to an EC2 instance using Jenkins.
