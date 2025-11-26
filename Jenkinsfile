pipeline {
    agent any

    environment {
        DEPLOY_DIR = '/opt/orbstore'
        PORT = '8088'

        // ==== DB DETAILS (NO CREDENTIALS USED) ====
        DB_HOST = "CHANGE_ME_DB_SERVER_IP"       // e.g. 172.31.25.100
        DB_NAME = "CHANGE_ME_DATABASE_NAME"      // e.g. clicknbuydb
        DB_USER = "CHANGE_ME_DB_USERNAME"        // e.g. clicknbuy_user
        DB_PASS = "CHANGE_ME_DB_PASSWORD"        // e.g. test123
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo "Building..."
                sh "mvn -DskipTests clean package"
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    set -e

                    echo "Checking JAR..."
                    JAR_FILE=$(ls target/*.jar | grep -v original | head -n1)
                    echo "Found JAR: $JAR_FILE"

                    echo "Killing old process on port ${PORT}..."
                    sudo kill -9 $(sudo lsof -t -i:${PORT}) 2>/dev/null || true

                    echo "Deploying..."
                    sudo mkdir -p ${DEPLOY_DIR}
                    sudo cp $JAR_FILE ${DEPLOY_DIR}/app.jar
                    sudo chown jenkins:jenkins ${DEPLOY_DIR}/app.jar

                    echo "Starting app..."
                    nohup sudo -u jenkins java -jar ${DEPLOY_DIR}/app.jar \
                        --server.port=${PORT} \
                        --spring.datasource.url=jdbc:mysql://${DB_HOST}:3306/${DB_NAME} \
                        --spring.datasource.username=${DB_USER} \
                        --spring.datasource.password=${DB_PASS} \
                        > ${DEPLOY_DIR}/app.log 2>&1 &
                    
                    sleep 6

                    echo "Checking app..."
                    ss -tulpn | grep :${PORT} || { 
                        echo "Start failed. Showing logs:"
                        sudo tail -n 100 ${DEPLOY_DIR}/app.log
                        exit 1
                    }

                    echo "Application started successfully on port ${PORT}"
                '''
            }
        }
    }

    post {
        failure {
            echo "Pipeline failed — check logs."
        }
        success {
            echo "Pipeline success!"
        }
    }
}
