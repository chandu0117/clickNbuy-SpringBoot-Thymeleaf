pipeline {
  agent any
  environment {
    DEPLOY_DIR = '/opt/orbstore'
    JAR_NAME = 'app.jar'
    PORT = '8088'

    // ==== REPLACE THESE WITH YOUR REAL DB VALUES ====
    DB_HOST = "CHANGE_ME_DB_SERVER_IP"     // e.g. 172.31.45.120
    DB_NAME = "CHANGE_ME_DB_NAME"          // e.g. clicknbuydb
    DB_USER = "CHANGE_ME_DB_USER"          // e.g. clicknbuy_user
    DB_PASS = "CHANGE_ME_DB_PASS"          // e.g. StrongP@ssw0rd
    // =================================================
  }
  stages {
    stage('Checkout') {
      steps {
        echo "Checking out repository..."
        checkout scm
      }
    }

    stage('Build') {
      steps {
        echo "Building with Maven (skip tests)..."
        sh 'mvn -B -DskipTests clean package'
      }
    }

    stage('Prepare') {
      steps {
        echo "Preparing deploy directory..."
        sh '''
          set -e
          sudo mkdir -p ${DEPLOY_DIR}
          sudo chown jenkins:jenkins ${DEPLOY_DIR} || true
          sudo chmod 750 ${DEPLOY_DIR} || true
        '''
      }
    }

    stage('Deploy') {
      steps {
        echo "Deploy: find jar and deploy"
        sh '''
          set -e
          # find the built jar (ignore *-original)
          JAR_FILE=$(ls target/*.jar 2>/dev/null | grep -v 'original' | head -n1 || true)
          if [ -z "$JAR_FILE" ]; then
            echo "ERROR: No jar found in target/"; ls -la target || exit 1
          fi
          echo "Found JAR: $JAR_FILE"

          # Wait for DB reachable (simple TCP check, 12 tries x 5s = 60s)
          if [ -n "${DB_HOST}" ]; then
            tries=0
            maxtries=12
            echo "Waiting for DB ${DB_HOST}:3306 to be reachable..."
            while ! nc -z ${DB_HOST} 3306; do
              tries=$((tries+1))
              if [ "${tries}" -ge "${maxtries}" ]; then
                echo "ERROR: DB at ${DB_HOST}:3306 not reachable after ${tries} tries"
                exit 2
              fi
              echo "DB not ready (try ${tries}/${maxtries}) — sleeping 5s"
              sleep 5
            done
            echo "DB reachable"
          else
            echo "DB_HOST empty; using internal app config"
          fi

          # kill any process binding to the port (safe)
          echo "Checking port ${PORT}..."
          PIDS="$(ss -tulpn | awk '/:'${PORT}'\\b/ {print $6}' | sed -E 's/.*pid=([0-9]+),.*/\\1/' | tr '\\n' ' ' | sed 's/^ *//;s/ *$//')"
          if [ -n "$PIDS" ]; then
            echo "Killing PIDs on port ${PORT}: $PIDS"
            for pid in $PIDS; do sudo kill -9 $pid || true; done
            echo "Cleaning ${DEPLOY_DIR}"
            sudo rm -rf ${DEPLOY_DIR}/*
          else
            echo "No process on ${PORT}"
          fi

          # copy jar to deploy dir
          sudo cp "$JAR_FILE" ${DEPLOY_DIR}/${JAR_NAME}
          sudo chown jenkins:jenkins ${DEPLOY_DIR}/${JAR_NAME}
          sudo chmod 640 ${DEPLOY_DIR}/${JAR_NAME}

          # Prepare DB overrides (will be appended to java command)
          DB_OVERRIDES="--spring.datasource.url=jdbc:mysql://${DB_HOST}:3306/${DB_NAME} --spring.datasource.username=${DB_USER} --spring.datasource.password=${DB_PASS}"

          # If systemd service exists, restart it; otherwise start with nohup
          if [ -f /etc/systemd/system/orb-app.service ]; then
            echo "Systemd service found — restarting orb-app"
            sudo systemctl daemon-reload || true
            sudo systemctl restart orb-app || sudo systemctl start orb-app
            sleep 5
            sudo systemctl is-active --quiet orb-app || (echo "systemd start failed; journal:"; sudo journalctl -u orb-app -n 200 --no-pager; exit 3)
          else
            echo "Starting app with nohup as jenkins user"
            nohup sudo -u jenkins sh -c "exec java -jar ${DEPLOY_DIR}/${JAR_NAME} --server.port=${PORT} ${DB_OVERRIDES}" > ${DEPLOY_DIR}/app.log 2>&1 &
            sleep 8
            if ss -tulpn | grep -q ":${PORT}\\b"; then
              echo "Application started and listening on ${PORT}"
            else
              echo "Application failed to start - last 200 log lines:"
              sudo -u jenkins tail -n 200 ${DEPLOY_DIR}/app.log || true
              exit 4
            fi
          fi

          echo "Deployment complete"
        '''
      }
    } // end Deploy
  } // end stages

  post {
    success {
      echo "Pipeline succeeded."
    }
    failure {
      echo "Pipeline failed — check console + /opt/orbstore/app.log"
    }
  }
}
