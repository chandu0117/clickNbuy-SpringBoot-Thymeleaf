pipeline {
  agent any
  environment {
    DEPLOY_DIR = '/opt/orbstore'
    JAR_NAME = 'app.jar'
    PORT = '8088'
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build') {
      steps {
        sh 'mvn -B -DskipTests clean package'
      }
    }

    stage('Deploy') {
      steps {
        sh '''
          set -e
          echo "Preparing deploy dir: ${DEPLOY_DIR}"
          sudo mkdir -p ${DEPLOY_DIR}
          sudo chown jenkins:jenkins ${DEPLOY_DIR} || true

          # find built jar
          JAR_FILE=$(ls target/*.jar 2>/dev/null | grep -v 'original' | head -n1 || true)
          if [ -z "$JAR_FILE" ]; then
            echo "ERROR: No jar found in target/"; ls -la target || exit 1
          fi
          echo "Found jar: $JAR_FILE"

          # If port is in use -> kill processes and clean deploy dir
          if ss -tulpn | grep -q ":${PORT}"; then
            echo "Port ${PORT} is in use — stopping processes"
            PIDS=$(ss -tulpn | awk "/:${PORT}/ {print \$6}" | sed -E 's/.*pid=([0-9]+),.*/\\1/' | tr "\\n" " ")
            echo "Killing pids: $PIDS"
            for pid in $PIDS; do
              sudo kill -9 $pid || true
            done
            echo "Removing existing files in ${DEPLOY_DIR}"
            sudo rm -rf ${DEPLOY_DIR}/*
          else
            echo "Port ${PORT} not in use — fresh deploy"
          fi

          # copy jar and set ownership
          sudo cp "$JAR_FILE" ${DEPLOY_DIR}/${JAR_NAME}
          sudo chown jenkins:jenkins ${DEPLOY_DIR}/${JAR_NAME}

          # if systemd service exists, restart it, else run via nohup
          if [ -f /etc/systemd/system/orb-app.service ]; then
            echo "Using systemd to manage orb-app"
            sudo systemctl daemon-reload || true
            sudo systemctl restart orb-app || sudo systemctl start orb-app
            sleep 5
            sudo systemctl is-active --quiet orb-app && echo "orb-app is active" || (sudo journalctl -u orb-app -n 200 && exit 1)
          else
            echo "Starting jar with nohup as jenkins user"
            nohup sudo -u jenkins java -jar ${DEPLOY_DIR}/${JAR_NAME} --server.port=${PORT} > ${DEPLOY_DIR}/app.log 2>&1 &
            sleep 5
            if ss -tulpn | grep -q ":${PORT}"; then
              echo "Application started on port ${PORT}"
            else
              echo "Failed to start app; show last 200 log lines:"
              sudo tail -n 200 ${DEPLOY_DIR}/app.log
              exit 1
            fi
          fi
        '''
      }
    }

    stage('Post') {
      steps {
        echo 'Pipeline finished successfully.'
      }
    }
  }

  post {
    failure {
      echo 'Pipeline failed — check console output and /opt/orbstore/app.log'
    }
  }
}
