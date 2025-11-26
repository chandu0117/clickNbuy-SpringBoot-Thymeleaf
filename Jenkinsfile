pipeline {
    agent any

    environment {
        APP_HOME = "/opt/springapp"
        APP_JAR  = "clickNbuy-app.jar"
    }

    stages {
        stage('Checkout') {
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
                // Create app directory if missing
                sh "sudo mkdir -p ${APP_HOME}"
                sh "sudo cp target/*.jar ${APP_HOME}/${APP_JAR}"
                sh "sudo cp src/main/resources/application.properties ${APP_HOME}/application.properties"
            }
        }

        stage('Run Application') {
            steps {
                // Stop if already running
                sh "sudo pkill -f ${APP_JAR} || true"
                
                // Start app in background
                sh "nohup java -jar ${APP_HOME}/${APP_JAR} --spring.config.location=${APP_HOME}/application.properties > ${APP_HOME}/app.log 2>&1 &"
            }
        }
    }

    post {
        success {
            echo "Application deployed and running on port 8088"
        }
        failure {
            echo "Pipeline failed! Check logs."
        }
    }
}
