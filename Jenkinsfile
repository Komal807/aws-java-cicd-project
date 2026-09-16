pipeline {
    agent any

    tools {
        jdk 'JAVA21'
        maven 'Maven3'
    }

    environment {
        JFROG_URL  = 'http://172.31.47.148:8082'
        JFROG_REPO = 'devops-libs-release-local'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Source code downloaded from GitHub'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarCloud') {
                    sh '''
                        mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                          -Dsonar.projectKey=Komal807_aws-java-cicd-project \
                          -Dsonar.organization=komal807
                    '''
                }
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }

        stage('Verify Artifact') {
            steps {
                sh 'ls -lh target/'
            }
        }

        stage('Upload to JFrog') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'jfrog-credentials',
                        usernameVariable: 'JFROG_USER',
                        passwordVariable: 'JFROG_TOKEN'
                    )
                ]) {
                    sh '''
                        curl -f \
                          -u "$JFROG_USER:$JFROG_TOKEN" \
                          -T target/aws-devops-app-1.0.0.jar \
                          "$JFROG_URL/artifactory/$JFROG_REPO/com/abhi/aws-devops-app/1.0.0/aws-devops-app-1.0.0.jar"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI Pipeline completed successfully!'
            echo 'SonarQube analysis completed and artifact uploaded to JFrog.'
        }

        failure {
            echo 'CI Pipeline failed!'
        }
    }
}
