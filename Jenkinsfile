pipeline {
    agent any

    environment {
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
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    mkdir -p ${APP_DIR}
                    chown jenkins:jenkins ${APP_DIR}
                    pkill -f ${JAR_NAME} || true
                    cp target/*.jar ${APP_DIR}/${JAR_NAME}
                    nohup java -jar ${APP_DIR}/${JAR_NAME} > ${APP_DIR}/app.log 2>&1 &
                """
            }
        }

        stage('Verify') {
            steps {
                sh 'ps -ef | grep clickNbuy-app.jar || echo "App is not running"'
                sh 'tail -n 10 /opt/springapp/app.log || echo "No log yet"'
            }
        }
    }
}
