pipeline {
    agent any

    tools {
        jdk 'JAVA21'
        maven 'Maven3'
    }

    environment {
        JFROG_URL  = 'http://172.31.47.148:8082'
        JFROG_REPO = 'devops-libs-release-local'
        APP_SERVER = '172.31.28.128'
        APP_DIR    = '/opt/aws-devops-app'
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

        stage('Download Artifact from JFrog') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'jfrog-credentials',
                        usernameVariable: 'JFROG_USER',
                        passwordVariable: 'JFROG_TOKEN'
                    )
                ]) {
                    sh '''
                        rm -f /tmp/aws-devops-app-1.0.0.jar

                        curl -f \
                          -u "$JFROG_USER:$JFROG_TOKEN" \
                          -o /tmp/aws-devops-app-1.0.0.jar \
                          "$JFROG_URL/artifactory/$JFROG_REPO/com/abhi/aws-devops-app/1.0.0/aws-devops-app-1.0.0.jar"
                    '''
                }
            }
        }

        stage('Deploy to App Server') {
            steps {
                sh '''
                    echo "Copying application to App Server..."

                    scp /tmp/aws-devops-app-1.0.0.jar \
                      ec2-user@$APP_SERVER:$APP_DIR/aws-devops-app-1.0.0.jar.new

                    echo "Stopping old application..."

                    ssh ec2-user@$APP_SERVER \
                      "pkill -f 'aws-devops-app-1.0.0.jar' || true"

                    echo "Installing new application..."

                    ssh ec2-user@$APP_SERVER \
                      "mv $APP_DIR/aws-devops-app-1.0.0.jar.new $APP_DIR/aws-devops-app-1.0.0.jar"

                    echo "Starting new application..."

                    ssh ec2-user@$APP_SERVER \
                      "cd $APP_DIR && nohup java -jar aws-devops-app-1.0.0.jar > app.log 2>&1 < /dev/null &"
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    echo "Waiting for application to start..."
                    sleep 10

                    ssh ec2-user@$APP_SERVER \
                      "curl -f http://localhost:8080/"
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD Pipeline completed successfully!'
            echo 'SonarQube analysis completed.'
            echo 'Artifact uploaded to JFrog.'
            echo 'Application deployed successfully to AWS EC2.'
        }

        failure {
            echo 'CI/CD Pipeline failed!'
        }
    }
}
