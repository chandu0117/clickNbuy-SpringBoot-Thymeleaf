pipeline {
  agent any
  environment {
    DEPLOY_DIR = '/opt/orbstore'
    JAR_NAME = 'app.jar'
    PORT = '8088'
    DB_WAIT_RETRIES = '12'    // 12 * 5s = 60s total wait
    DB_WAIT_SLEEP = '5'       // seconds between checks
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build') {
      steps {
        echo "Building with Maven..."
        sh 'mvn -B -DskipTests clean package'
      }
    }

    stage('Prepare') {
      steps {
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
        withCredentials([
           usernamePassword(credentialsId: 'db-creds', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS'),
           string(credentialsId: 'db-host', variable: 'DB_HOST'),
           string(credentialsId: 'db-name', variable: 'DB_NAME')
        ]) {
          sh '''
            set -e
            echo "Searching for built JAR..."
            JAR_FILE=$(ls target/*.jar 2>/dev/null | grep -v 'original' | head -n1 || true)
            if [ -z "$JAR_FILE" ]; then
              echo "No JAR in target/ - aborting"; ls -la target || true; exit 1
            fi
            echo "JAR found: $JAR_FILE"

            # wait for DB reachable (simple TCP check)
            if [ -n "$DB_HOST" ]; then
              echo "Waiting for DB ${DB_HOST} (up to $((DB_WAIT_RETRIES*DB_WAIT_SLEEP))s)..."
              tries=0
              while ! nc -z ${DB_HOST} 3306; do
                tries=$((tries+1))
                if [ "$tries" -ge ${DB_WAIT_RETRIES} ]; then
                  echo "DB at ${DB_HOST}:3306 not reachable after ${tries} tries"; exit 2
                fi
                echo "DB not ready - sleeping ${DB_WAIT_SLEEP}s (try ${tries}/${DB_WAIT_RETRIES})"
                sleep ${DB_WAIT_SLEEP}
              done
              echo "DB reachable"
            else
              echo "No DB_HOST provided, will use app internal config"
            fi

            # if port in use -> kill it
            if ss -tulpn | grep -q ":${PORT}\\b"; then
              echo "Port ${PORT} in use - killing previous processes"
              PIDS=$(ss -tulpn | awk "/:${PORT}\\b/ {print \$6}" | sed -E 's/.*pid=([0-9]+),.*/\\1/' | tr "\\n" " " || true)
              for pid in $PIDS; do
                if [ -n "$pid" ]; then
                  echo "Killing pid $pid"; sudo kill -9 $pid || true
                fi
              done
              echo "Cleaning ${DEPLOY_DIR}"
              sudo rm -rf ${DEPLOY_DIR}/*
            else
              echo "No process on ${PORT}"
            fi

            # copy jar to deploy dir
            sudo cp "$JAR_FILE" ${DEPLOY_DIR}/${JAR_NAME}
            sudo chown jenkins:jenkins ${DEPLOY_DIR}/${JAR_NAME}
            sudo chmod 640 ${DEPLOY_DIR}/${JAR_NAME}

            # build DB overrides if provided
            DB_OVERRIDES=""
            if [ -n "${DB_HOST}" ] && [ -n "${DB_NAME}" ]; then
              DB_OVERRIDES="--spring.datasource.url=jdbc:mysql://${DB_HOST}:3306/${DB_NAME} --spring.datasource.username=${DB_USER} --spring.datasource.password=${DB_PASS}"
              echo "DB overrides will be used"
            else
              echo "DB overrides not provided; using internal config"
            fi

            # start with systemd if exists else nohup
            if [ -f /etc/systemd/system/orb-app.service ]; then
              echo "Using systemd to manage app"
              sudo systemctl daemon-reload || true
              sudo systemctl restart orb-app || sudo systemctl start orb-app
              sleep 5
              sudo systemctl is-active --quiet orb-app || (echo "systemd failed - show journal"; sudo journalctl -u orb-app -n 200 --no-pager; exit 3)
            else
              echo "Starting with nohup as jenkins user"
              nohup sudo -u jenkins sh -c "exec java -jar ${DEPLOY_DIR}/${JAR_NAME} --server.port=${PORT} ${DB_OVERRIDES}" > ${DEPLOY_DIR}/app.log 2>&1 &
              sleep 8
              if ss -tulpn | grep -q ":${PORT}\\b"; then
                echo "Application listening on ${PORT}"
              else
                echo "Application failed to start - tailing logs"; sudo -u jenkins tail -n 200 ${DEPLOY_DIR}/app.log || true; exit 4
              fi
            fi

            echo "Deploy complete"
          '''
        } // end withCredentials
      } // end steps
    } // end Deploy
  } // end stages

  post {
    success {
      echo "Pipeline finished successfully."
    }
    failure {
      echo "Pipeline failed - attach console and /opt/orbstore/app.log for help."
    }
  }
}
