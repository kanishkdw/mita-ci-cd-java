// Jenkinsfile - manual SSH clone + flexible build (corrected)
pipeline {
  agent any

  environment {
    REPO_SSH        = 'git@github.com:kanishkdw/mita-ci-cd-java.git'
    SSH_KEY_PATH    = '/var/jenkins_home/.ssh/id_ed25519'
    KNOWN_HOSTS     = '/var/jenkins_home/.ssh/known_hosts'
    CLONE_DIR       = 'repo'
    SONAR_HOST_URL  = 'http://sonarqube:9000'
    DOCKER_REGISTRY = 'docker.io'
    IMAGE_NAME      = 'kanishkdw/mita-ci-cd-java'
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '20'))
    timestamps()
  }

  stages {
    stage('Cleanup') {
      steps {
        echo "Cleaning workspace"
        deleteDir()
      }
    }

    stage('Manual SSH clone') {
      steps {
        script {
          sh '''
            echo "Checking SSH key and known_hosts"
            ls -l "${SSH_KEY_PATH}" || true
            ls -l "${KNOWN_HOSTS}" || true
          '''
          sh '''
            set -e
            REPO="${REPO_SSH}"
            TARGET_DIR="${CLONE_DIR}"
            rm -rf "${TARGET_DIR}"
            mkdir -p "${TARGET_DIR}"
            echo "Cloning ${REPO} into ${TARGET_DIR} ..."
            GIT_SSH_COMMAND="ssh -i ${SSH_KEY_PATH} -o IdentitiesOnly=yes -o StrictHostKeyChecking=yes -o UserKnownHostsFile=${KNOWN_HOSTS} -v" \
              git clone --depth=1 "${REPO}" "${TARGET_DIR}" 2>&1 | sed -n '1,200p' || true
            if [ -d "${TARGET_DIR}/.git" ]; then
              echo "Clone appears successful. HEAD:"
              git --git-dir="${TARGET_DIR}/.git" rev-parse --verify HEAD || true
            else
              echo "Clone failed - repository directory not found."
              exit 1
            fi
          '''
        }
      }
    }

    stage('Inspect project') {
      steps {
        script {
          sh '''
            set -e
            cd "${CLONE_DIR}"
            echo "Files in top of repo:"
            ls -la | sed -n '1,200p'
            if [ -f pom.xml ]; then
              echo "PROJECT_TYPE=maven" > ../project_type.env
            elif [ -f package.json ]; then
              echo "PROJECT_TYPE=node" > ../project_type.env
            else
              echo "PROJECT_TYPE=unknown" > ../project_type.env
            fi
          '''
          def props = readFile('project_type.env').trim()
          echo "Detected: ${props}"
          env.PROJECT_TYPE = props.replace('PROJECT_TYPE=','').trim()
        }
      }
    }

    stage('Build') {
      steps {
        script {
          if (env.PROJECT_TYPE == 'maven') {
            echo "Running Maven build"
            sh '''
              cd "${CLONE_DIR}"
              if command -v mvn >/dev/null 2>&1; then
                mvn -B -DskipTests clean package
              else
                echo "Maven is not available on the agent. Please install mvn or run this on an agent having Maven."
                exit 1
              fi
            '''
          } else if (env.PROJECT_TYPE == 'node') {
            echo "Running Node build"
            sh '''
              cd "${CLONE_DIR}"
              if command -v npm >/dev/null 2>&1; then
                npm ci || npm install
                if grep -q "\"build\"" package.json; then
                  npm run build
                else
                  echo "No build script; listing files"
                  ls -la
                fi
              else
                echo "npm not found on agent - cannot build Node project"
                exit 1
              fi
            '''
          } else {
            echo "Unknown project type; no build step executed"
            sh '''
              cd "${CLONE_DIR}"
              echo "You can implement custom build steps here."
              ls -la
            '''
          }
        }
      }
    }

    stage('Test') {
      when {
        expression { env.PROJECT_TYPE == 'maven' }
      }
      steps {
        script {
          sh '''
            cd "${CLONE_DIR}"
            if command -v mvn >/dev/null 2>&1; then
              mvn -B test || true
            else
              echo "Maven not available for tests."
            fi
          '''
        }
      }
      post {
        always {
          junit "${CLONE_DIR}/target/surefire-reports/*.xml"
        }
      }
    }

    stage('Archive artifacts & test outputs') {
      steps {
        script {
          if (env.PROJECT_TYPE == 'maven') {
            sh 'ls -la ${CLONE_DIR}/target || true'
            archiveArtifacts artifacts: "${CLONE_DIR}/target/**/*.jar", allowEmptyArchive: true
          }
          if (env.PROJECT_TYPE == 'node') {
            archiveArtifacts artifacts: "${CLONE_DIR}/dist/**", allowEmptyArchive: true
            archiveArtifacts artifacts: "${CLONE_DIR}/build/**", allowEmptyArchive: true
          }
        }
      }
    }

    stage('SonarQube Analysis') {
      when {
        expression { env.PROJECT_TYPE == 'maven' }
      }
      steps {
        // use withCredentials to keep token out of Groovy logs
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
          sh '''
            cd "${CLONE_DIR}"
            if ! command -v mvn >/dev/null 2>&1; then
              echo "Maven not found - cannot run Sonar analysis"
              exit 1
            fi
            mvn -B sonar:sonar \
              -Dsonar.projectKey=mita-ci-cd-java \
              -Dsonar.host.url=$SONAR_HOST_URL \
              -Dsonar.login=$SONAR_TOKEN
          '''
        }
      }
    }

    stage('Deploy') {
      when {
        expression { env.PROJECT_TYPE == 'maven' }
      }
      steps {
        echo "Deploying MITA App..."
        sh '''
          cd "${CLONE_DIR}/target"
          JAR_FILE=$(ls *.jar | head -n 1 || true)
          if [ -z "$JAR_FILE" ]; then
            echo "No jar found to deploy"
            exit 1
          fi
          echo "Running $JAR_FILE ..."
          nohup java -jar "$JAR_FILE" > app.log 2>&1 &
          sleep 5
          echo "App deployed and running in background (app.log)."
        '''
      }
    }
    // add more stages here (docker build/push, integration tests, deploy to k8s, etc.)
  } // end stages

  post {
    success {
      echo "Pipeline finished SUCCESS"
    }
    failure {
      echo "Pipeline finished FAILURE"
    }
    always {
      echo "Workspace listing (top):"
      sh 'ls -la || true'
    }
  }
}
