pipeline {
    agent any
    stages {
        stage('Initialize Pipeline'){
            steps {
                echo 'Initializing Pipeline ...'
            }
        }
        stage('Checkout GitHub Codes'){
            steps {
                echo 'Checking out GitHub Codes'
            }
        }
        stage('Maven Build'){
            steps {
                echo 'Building Java App with Maven'
            }
        }
        stage('JUnit Test of Java App'){
            steps {
                echo 'JUnit Testing'
            }
        }
        stage('SonarQube Analysis'){
            steps {
                echo 'Running Static Code Analysis with SonarQube'
            }
        }
        stage('Trivy FS Scan'){
            steps {
                echo 'Scanning File System with Trivy FS ...'
            }
        }
        stage('Build & Tag Docker Image'){
            steps {
                echo 'Building the Java App Docker Image'
            }
        }
        stage('Trivy Security Scan'){
            steps {
                echo 'Scanning Docker Image with Trivy'
            }
        }
		stage('Authenticate with GCP, Tag & Push to Artifact Registry') {
            steps {
				echo 'Authenticate with GCP, tag and Push Image to Artifact Registry'
            }
        }
	    stage('Deploy to Cloud Run') {
		    steps {
				echo 'Deploying Image to Google Cloud Run'
		    }
	    }
	    stage('Get Cloud Run Service URL') {
            steps {
				echo 'Getting Cloud Run Service URL'
            }
        }
	}
}
