// Declarative Pipeline Script for Java/Maven CI/CD
// This pipeline includes: Checkout, Build, Preparation (directory setup), 
// and a robust Deployment with DB reachability check and application restart.

pipeline {
    agent any

    environment {
        // --- Deployment Configuration ---
        DEPLOY_DIR = '/opt/orbstore'
        JAR_NAME = 'app.jar'
        PORT = '8088'

        // === FILL THESE WITH YOUR VALUES (IMPORTANT: Use Jenkins Credentials for sensitive data if possible) ===
        // NOTE: If using the Jenkins Credential Manager, you would store these as Secret Text
        // and inject them as environment variables using the 'withCredentials' wrapper in the deploy stage.
        DB_HOST = "172.31.45.120"
        DB_NAME = "clicknbuydb"
        DB_USER = "clicknbuy_user"
        DB_PASS = "StrongP@ssw0rd"
        // ===================================
    }

    stages {
        stage('Checkout Private Repository') {
            steps {
                echo "Cloning source code from the private repository..."
                // FIX: Replaced the placeholder URL with the actual repository URL and changed branch to 'master'.
                script {
                    git branch: 'master', credentialsId: 'private-repo-ssh-key', url: 'https://github.com/chandu0117/clickNbuy-SpringBoot-Thymeleaf.git'
                }
            }
        }

        stage('Build Project') {
            steps {
                echo 'Building the project using Maven and skipping tests...'
                sh 'mvn -B -DskipTests clean package'
            }
        }

        stage('Prepare Deployment Directory') {
            steps {
                echo "Ensuring deployment directory ${DEPLOY_DIR} exists and permissions are correct."
                sh '''
                    # Create directory if it doesn't exist
                    sudo mkdir -p ${DEPLOY_DIR}
                    # Set ownership to 'jenkins' user to allow app to write logs, etc.
                    # '|| true' prevents failure if the user is not found or chown fails for other reasons, though it should ideally succeed.
                    sudo chown jenkins:jenkins ${DEPLOY_DIR} || true 
                '''
            }
        }

        stage('Deploy & Restart Application') {
            steps {
                echo 'Starting robust deployment sequence: DB check, kill old process, copy JAR, start new app.'
                sh '''
                    # Exit immediately if a command exits with a non-zero status
                    set -e 
                    
                    # 1. Find the generated JAR file, excluding common secondary artifacts
                    JAR_FILE=$(ls target/*.jar 2>/dev/null | grep -v original | head -n1 || true)
                    if [ -z "$JAR_FILE" ]; then 
                        echo "FATAL ERROR: Application JAR file not found in target directory."
                        ls -la target
                        exit 1 
                    fi

                    # 2. Database Reachability Check (Essential Pre-Deployment Step)
                    echo "Checking database connection to ${DB_HOST}:3306..."
                    tries=0; max=12
                    while ! nc -z ${DB_HOST} 3306; do
                        tries=$((tries+1))
                        if [ "$tries" -ge "$max" ]; then 
                            echo "ERROR: Database ${DB_HOST} not reachable after $((max*5)) seconds." 
                            exit 2 
                        fi
                        echo "DB not reachable (Try $tries/$max). Waiting 5s..."
                        sleep 5
                    done
                    echo "Database connection successful."

                    # 3. Kill Previous Application Process
                    echo "Stopping any existing application on port ${PORT}..."
                    # Use 'ss' (Socket Statistics) to find the PID listening on the port.
                    PIDS="$(ss -tulpn | awk '/:'${PORT}'\\b/ {print $6}' | sed -E 's/.*pid=([0-9]+),.*/\\1/' | tr '\\n' ' ')"
                    for p in $PIDS; do 
                        echo "Killing PID: $p..."
                        sudo kill -9 $p || true 
                    done
                    # Clean up old deployment files
                    sudo rm -rf ${DEPLOY_DIR}/* || true

                    # 4. Copy New JAR
                    echo "Copying $JAR_FILE to deployment directory ${DEPLOY_DIR}/${JAR_NAME}..."
                    sudo cp "$JAR_FILE" ${DEPLOY_DIR}/${JAR_NAME}
                    sudo chown jenkins:jenkins ${DEPLOY_DIR}/${JAR_NAME}

                    # 5. Start New Application
                    echo "Starting new application as 'jenkins' user..."
                    # 'nohup sudo -u jenkins sh -c "exec java -jar..."': The most robust way to start a background process as a specific user.
                    # Spring Boot config is passed via command line arguments.
                    nohup sudo -u jenkins sh -c "exec java -jar ${DEPLOY_DIR}/${JAR_NAME} --server.port=${PORT} --spring.datasource.url=jdbc:mysql://${DB_HOST}:3306/${DB_NAME} --spring.datasource.username=${DB_USER} --spring.datasource.password=${DB_PASS}" > ${DEPLOY_DIR}/app.log 2>&1 &
                    
                    # 6. Basic Health Check
                    echo "Waiting 8 seconds for application to start..."
                    sleep 8
                    if ss -tulpn | grep -q ":${PORT}\\b"; then 
                        echo "SUCCESS: Application started successfully on port ${PORT}." 
                    else 
                        echo "FAILURE: Application did not start. Displaying logs:"
                        # Show the last 200 lines of the application log for debugging
                        sudo -u jenkins tail -n 200 ${DEPLOY_DIR}/app.log
                        exit 3 
                    fi
                '''
            }
        }
    }

    post {
        failure { 
            echo "Pipeline failed — check the Jenkins console output and the application log at ${DEPLOY_DIR}/app.log for details." 
        }
        success { 
            echo "Pipeline succeeded! Application is running on port ${PORT}." 
        }
    }
}
