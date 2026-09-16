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

        APP_NAME   = 'aws-devops-app-1.0.0.jar'
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
                sh '''
                    echo "Verifying generated artifact..."
                    ls -lh target/
                    test -f target/$APP_NAME
                '''
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
                        echo "Uploading artifact to JFrog..."

                        curl -f \
                          -u "$JFROG_USER:$JFROG_TOKEN" \
                          -T target/$APP_NAME \
                          "$JFROG_URL/artifactory/$JFROG_REPO/com/abhi/aws-devops-app/1.0.0/$APP_NAME"
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
                        echo "Downloading deployment artifact from JFrog..."

                        rm -f /tmp/$APP_NAME

                        curl -f \
                          -u "$JFROG_USER:$JFROG_TOKEN" \
                          -o /tmp/$APP_NAME \
                          "$JFROG_URL/artifactory/$JFROG_REPO/com/abhi/aws-devops-app/1.0.0/$APP_NAME"

                        echo "Artifact downloaded successfully."

                        ls -lh /tmp/$APP_NAME
                    '''
                }
            }
        }

        stage('Deploy to App Server') {
            steps {
                sh '''
                    echo "======================================"
                    echo "Deploying application to AWS EC2"
                    echo "======================================"

                    echo "1. Copying new artifact to App Server..."

                    scp /tmp/$APP_NAME \
                      ec2-user@$APP_SERVER:$APP_DIR/$APP_NAME.new


                    echo "2. Stopping previous application..."

                    ssh ec2-user@$APP_SERVER "
                        if [ -f $APP_DIR/app.pid ]; then

                            OLD_PID=\$(cat $APP_DIR/app.pid)

                            if kill -0 \$OLD_PID 2>/dev/null; then
                                echo Stopping PID \$OLD_PID
                                kill \$OLD_PID
                                sleep 5
                            fi

                            rm -f $APP_DIR/app.pid
                        else
                            echo No PID file found. Continuing deployment.
                        fi
                    "


                    echo "3. Installing new application..."

                    ssh ec2-user@$APP_SERVER \
                      "mv $APP_DIR/$APP_NAME.new $APP_DIR/$APP_NAME"


                    echo "4. Starting new application..."

                    ssh ec2-user@$APP_SERVER "
                        cd $APP_DIR

                        nohup java -jar $APP_NAME \
                          > app.log 2>&1 < /dev/null &

                        echo \\\$! > app.pid
                    "


                    echo "5. Deployment command completed."
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    echo "Waiting for Spring Boot application to start..."

                    sleep 15

                    echo "Checking application..."

                    ssh ec2-user@$APP_SERVER \
                      "curl --fail --silent --show-error http://localhost:8080/"

                    echo ""
                    echo "Application health check successful."
                '''
            }
        }
    }

    post {

        success {
            echo '========================================='
            echo 'CI/CD PIPELINE SUCCESSFUL'
            echo '========================================='
            echo 'GitHub source checkout completed.'
            echo 'Maven build completed.'
            echo 'SonarQube analysis completed.'
            echo 'Artifact uploaded to JFrog.'
            echo 'Artifact downloaded from JFrog.'
            echo 'Application deployed to AWS EC2.'
            echo 'Application health check passed.'
        }

        failure {
            echo '========================================='
            echo 'CI/CD PIPELINE FAILED'
            echo '========================================='
            echo 'Check the failed Jenkins stage above.'
        }
    }
}
