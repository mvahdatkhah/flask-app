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
