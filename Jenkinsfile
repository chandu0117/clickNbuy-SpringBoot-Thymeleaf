pipeline {
  agent any
  environment {
    DEPLOY_DIR = '/opt/orbstore'
    JAR_NAME = 'app.jar'
    PORT = '8088'
    // Repo and branch are taken from job config (Pipeline script from SCM)
  }
  stages {
    stage('Checkout') {
      steps {
        // 'checkout scm' will use the repository and credentials from job config
        checkout scm
      }
    }

    stage('Build') {
      steps {
        echo "Building with Maven..."
        sh 'mvn -B -DskipTests clean package'
      }
    }

    stage('Deploy') {
      // inject DB credentials securely
      steps {
        withCredentials([
          usernamePassword(credentialsId: 'db-creds', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS'),
          string(credentialsId: 'db-host', variable: 'DB_HOST'),
          string(credentialsId: 'db-name', variable: 'DB_NAME')
        ]) {
          sh '''
            set -e
            echo "Preparing deploy dir: ${DEPLOY_DIR}"
            sudo mkdir -p ${DEPLOY_DIR}
            sudo chown jenkins:jenkins ${DEPLOY_DIR} || true

            # find built jar (ignore -original)
            JAR_FILE=$(ls target/*.jar 2>/dev/null | grep -v 'original' | head -n1 || true)
            if [ -z "$JAR_FILE" ]; then
              echo "ERROR: No JAR found in target/"; ls -la target; exit 1
            fi
            echo "Found JAR: $JAR_FILE"

            # if port in use -> kill processes and clean deploy dir
            if ss -tulpn | grep -q ":${PORT}\\b"; then
              echo "Port ${PORT} in use — killing related processes"
              PIDS=$(ss -tulpn | awk "/:${PORT}\\b/ {print \$6}" | sed -E 's/.*pid=([0-9]+),.*/\\1/' | tr "\\n" " ")
              echo "PIDs to kill: $PIDS"
              for pid in $PIDS; do
                sudo kill -9 $pid || true
              done
              echo "Cleaning ${DEPLOY_DIR}"
              sudo rm -rf ${DEPLOY_DIR}/*
            else
              echo "Port ${PORT} not in use — fresh deploy"
            fi

            # copy jar and set ownership
            sudo cp "$JAR_FILE" ${DEPLOY_DIR}/${JAR_NAME}
            sudo chown jenkins:jenkins ${DEPLOY_DIR}/${JAR_NAME}

            # build DB override string if DB_HOST/DB_NAME provided
            DB_OVERRIDES=""
            if [ -n "${DB_HOST}" ] && [ -n "${DB_NAME}" ]; then
              DB_OVERRIDES="--spring.datasource.url=jdbc:mysql://${DB_HOST}:3306/${DB_NAME} --spring.datasource.username=${DB_USER} --spring.datasource.password=${DB_PASS}"
            else
              echo "DB_HOST or DB_NAME not provided — using app's internal configuration"
            fi

            # restart logic: systemd preferred
            if [ -f /etc/systemd/system/orb-app.service ]; then
              echo "Restarting orb-app via systemd"
              sudo systemctl daemon-reload || true
              sudo systemctl restart orb-app || sudo systemctl start orb-app
              sleep 5
              if sudo systemctl is-active --quiet orb-app; then
                echo "orb-app active"
              else
                echo "systemd start failed - journal (last 200 lines):"
                sudo journalctl -u orb-app -n 200 --no-pager
                exit 1
              fi
            else
              echo "Starting jar with nohup as jenkins user"
              # kill background java already done above. Start new process with DB overrides if any
              nohup sudo -u jenkins sh -c "exec java -jar ${DEPLOY_DIR}/${JAR_NAME} --server.port=${PORT} ${DB_OVERRIDES}" > ${DEPLOY_DIR}/app.log 2>&1 &
              sleep 6
              if ss -tulpn | grep -q ":${PORT}\\b"; then
                echo "Application started on port ${PORT}"
              else
                echo "Failed to start - show last 200 log lines:"
                sudo -u jenkins tail -n 200 ${DEPLOY_DIR}/app.log || true
                exit 1
              fi
            fi
          '''
        } // end withCredentials
      } // end steps
    } // end Deploy

    stage('Post') {
      steps {
        echo "Deployment stage completed."
      }
    }
  } // end stages

  post {
    success {
      echo "Pipeline succeeded."
    }
    failure {
      echo "Pipeline failed — check console output and ${DEPLOY_DIR}/app.log"
    }
  }
}
