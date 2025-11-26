pipeline {
  agent any
  environment {
    DEPLOY_DIR = '/opt/orbstore'
    JAR_NAME = 'app.jar'
    PORT = '8088'

    // === FILL THESE WITH YOUR VALUES ===
    DB_HOST = "172.31.45.120"
    DB_NAME = "clicknbuydb"
    DB_USER = "clicknbuy_user"
    DB_PASS = "StrongP@ssw0rd"
    // ===================================
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build') {
      steps { sh 'mvn -B -DskipTests clean package' }
    }

    stage('Prepare') {
      steps {
        sh '''
          sudo mkdir -p ${DEPLOY_DIR}
          sudo chown jenkins:jenkins ${DEPLOY_DIR} || true
        '''
      }
    }

    stage('Deploy') {
      steps {
        sh '''
          set -e
          JAR_FILE=$(ls target/*.jar 2>/dev/null | grep -v original | head -n1 || true)
          if [ -z "$JAR_FILE" ]; then echo "No JAR found"; ls -la target; exit 1; fi

          # wait for DB reachable (12 tries * 5s)
          tries=0; max=12
          while ! nc -z ${DB_HOST} 3306; do
            tries=$((tries+1))
            if [ "$tries" -ge "$max" ]; then echo "DB not reachable"; exit 2; fi
            sleep 5
          done

          # kill previous app
          PIDS="$(ss -tulpn | awk '/:'${PORT}'\\b/ {print $6}' | sed -E 's/.*pid=([0-9]+),.*/\\1/' | tr '\\n' ' ')"
          for p in $PIDS; do sudo kill -9 $p || true; done
          sudo rm -rf ${DEPLOY_DIR}/* || true

          sudo cp "$JAR_FILE" ${DEPLOY_DIR}/${JAR_NAME}
          sudo chown jenkins:jenkins ${DEPLOY_DIR}/${JAR_NAME}

          nohup sudo -u jenkins sh -c "exec java -jar ${DEPLOY_DIR}/${JAR_NAME} --server.port=${PORT} --spring.datasource.url=jdbc:mysql://${DB_HOST}:3306/${DB_NAME} --spring.datasource.username=${DB_USER} --spring.datasource.password=${DB_PASS}" > ${DEPLOY_DIR}/app.log 2>&1 &

          sleep 8
          if ss -tulpn | grep -q ":${PORT}\\b"; then echo "Started"; else sudo -u jenkins tail -n 200 ${DEPLOY_DIR}/app.log; exit 3; fi
        '''
      }
    }
  }
  post {
    failure { echo "Pipeline failed — check console + /opt/orbstore/app.log" }
    success { echo "Pipeline succeeded" }
  }
}
