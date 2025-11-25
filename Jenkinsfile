pipeline {
    agent any

    tools {
        // Make sure these match the names you configured in Jenkins → Global Tool Configuration
        maven 'MAVEN_HOME'
        jdk 'JAVA_HOME'
    }

    environment {
        // Path to deploy your Spring Boot JAR
        APP_DIR = '/opt/springapp'
        JAR_NAME = 'clickNbuy-app.jar'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/chandu0117/clickNbuy-SpringBoot-Thymeleaf.git',
                    credentialsId: 'github-jenkins'
            }
        }

        stage('Build') {
            steps {
                echo "Building the Spring Boot project with Maven..."
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Deploy') {
            steps {
                echo "Deploying application..."
                sh """
                    # Create app directory if not exists
                    mkdir -p ${APP_DIR}
                    chown jenkins:jenkins ${APP_DIR}

                    # Stop old app if running
                    pkill -f ${JAR_NAME} || true

                    # Copy new JAR
                    cp target/*.jar ${APP_DIR}/${JAR_NAME}

                    # Start app in background
                    nohup java -jar ${APP_DIR}/${JAR_NAME} > ${APP_DIR}/app.log 2>&1 &
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                echo "Verifying deployment..."
                sh 'ps -ef | grep clickNbuy-app.jar || echo "App is not running"'
                sh 'tail -n 10 /opt/springapp/app.log || echo "No log yet"'
            }
        }
    }

    post {
        success {
            echo 'Pipeline finished successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
