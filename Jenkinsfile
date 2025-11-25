pipeline {
    agent any

    tools {
        maven 'MAVEN_HOME'  // Must match your Jenkins Maven config
        jdk 'JAVA_HOME'     // Must match your Jenkins JDK config
    }

    stages {

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                echo "Stopping old app if running..."
                pkill -f "clickNbuy-app.jar" || true

                echo "Copying new JAR..."
                cp target/*.jar /opt/springapp/clickNbuy-app.jar

                echo "Starting application..."
                nohup java -jar /opt/springapp/clickNbuy-app.jar > /opt/springapp/app.log 2>&1 &
                '''
            }
        }
    }
}

