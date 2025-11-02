// Jenkinsfile - manual SSH clone + flexible build
pipeline {
  agent any

  environment {
    REPO_SSH = 'git@github.com:kanishkdw/mita-ci-cd-java.git'
    // Path inside the Jenkins container where the SSH key and known_hosts live
    SSH_KEY_PATH = '/var/jenkins_home/.ssh/id_ed25519'
    // Optional: known hosts path (we assume it exists)
    KNOWN_HOSTS = '/var/jenkins_home/.ssh/known_hosts'
    CLONE_DIR = 'repo'         // where repo will be cloned

    // SonarQube config (require you to add Jenkins credential 'sonar-token' secret text)
    SONAR_HOST_URL = 'http://host.docker.internal:9000' // adjust if needed
    SONAR_TOKEN = credentials('sonar-token')

    // Optional Docker push config (set DOCKER_REGISTRY and DOCKER_CREDENTIALS_ID in Jenkins if you want Docker push)
    DOCKER_REGISTRY = ''          // e.g. registry.hub.docker.com or ghcr.io/myorg (leave empty to skip Docker push)
    DOCKER_CREDENTIALS_ID = ''    // Jenkins credential id (username/password) for docker login
    IMAGE_NAME = "kanishkdw/mita-ci-cd-java" // default image name (change as needed)
  }

  options {
    // keep build logs for a while
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
          // ensure the SSH key is present and readable
          sh '''
            echo "Checking SSH key and known_hosts"
            ls -l "${SSH_KEY_PATH}" || true
            ls -l "${KNOWN_HOSTS}" || true
          '''

          // do the clone with the explicit SSH identity file
          // -o IdentitiesOnly=yes guarantees only the provided key is used
          // -o StrictHostKeyChecking=yes forces hostkey checking against known_hosts
          sh '''
            set -e
            REPO="${REPO_SSH}"
            TARGET_DIR="${CLONE_DIR}"
            # remove stale dir if any
            rm -rf "${TARGET_DIR}"
            mkdir -p "${TARGET_DIR}"
            echo "Cloning ${REPO} into ${TARGET_DIR} ..."
            GIT_SSH_COMMAND="ssh -i ${SSH_KEY_PATH} -o IdentitiesOnly=yes -o StrictHostKeyChecking=yes -o UserKnownHostsFile=${KNOWN_HOSTS} -v" \
              git clone --depth=1 "${REPO}" "${TARGET_DIR}" 2>&1 | sed -n '1,200p' || true
            # show HEAD to confirm success
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
            # detect project type
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
          // load into env.PROJECT_TYPE for later stages
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
              # run basic maven build (skip tests by default)
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
                # run build if script exists, otherwise just list files
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

    stage('Archive artifacts & test outputs') {
      steps {
        script {
          // archive jar targets if maven
          if (env.PROJECT_TYPE == 'maven') {
            sh 'ls -la ${CLONE_DIR}/target || true'
            archiveArtifacts artifacts: "${CLONE_DIR}/target/**/*.jar", allowEmptyArchive: true
          }
          // archive build folder if node
          if (env.PROJECT_TYPE == 'node') {
            archiveArtifacts artifacts: "${CLONE_DIR}/dist/**", allowEmptyArchive: true
            archiveArtifacts artifacts: "${CLONE_DIR}/build/**", allowEmptyArchive: true
          }
        }
      }
    }

    stage('Optional: run tests (example)') {
      when {
        expression { env.PROJECT_TYPE == 'maven' }
      }
      steps {
        sh '''
          cd "${CLONE_DIR}"
          if [ -f pom.xml ]; then
            mvn -B test || true
          fi
        '''
      }
    }

    // -------------------------
    // NEW / EXTRA STAGES (kept after your original pipeline)
    // -------------------------

    stage('Test (publish reports)') {
      when {
        expression { env.PROJECT_TYPE == 'maven' }
      }
      steps {
        echo 'Running unit tests (if not already run) and publishing JUnit reports...'
        script {
          // run tests if not run previously (we already ran in Optional stage, but re-run is OK if needed)
          sh '''
            cd "${CLONE_DIR}"
            if command -v mvn >/dev/null 2>&1; then
              mvn -B test || true
            else
              echo "maven not available - skipping test run"
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

    stage('SonarQube Analysis') {
      when {
        allOf {
          expression { env.PROJECT_TYPE == 'maven' }
          expression { return env.SONAR_TOKEN != null && env.SONAR_TOKEN != '' }
        }
      }
      steps {
        echo "Starting SonarQube analysis (project: mita-ci-cd-java)..."
        sh """
          cd "${CLONE_DIR}"
          if command -v mvn >/dev/null 2>&1; then
            mvn -B sonar:sonar \
              -Dsonar.projectKey=mita-ci-cd-java \
              -Dsonar.host.url=${SONAR_HOST_URL} \
              -Dsonar.login=${SONAR_TOKEN}
          else
            echo "Maven not found - cannot run SonarQube analysis"
            exit 1
          fi
        """
      }
    }

    stage('Docker Build & Push (optional)') {
      when {
        allOf {
          expression { return env.DOCKER_REGISTRY != null && env.DOCKER_REGISTRY != '' }
          expression { return env.DOCKER_CREDENTIALS_ID != null && env.DOCKER_CREDENTIALS_ID != '' }
        }
      }
      steps {
        script {
          echo "Building Docker image and pushing to ${DOCKER_REGISTRY} (optional)"
          withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh '''
              set -e
              cd "${CLONE_DIR}"
              # Tagging: registry/imagename:build-${BUILD_NUMBER}
              IMAGE_TAG="${DOCKER_REGISTRY}/${IMAGE_NAME}:build-${BUILD_NUMBER}"
              echo "Building ${IMAGE_TAG} ..."
              if [ -f Dockerfile ]; then
                docker build -t "${IMAGE_TAG}" .
              else
                echo "No Dockerfile found in repo; skipping docker build"
                exit 1
              fi
              echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin "${DOCKER_REGISTRY}"
              docker push "${IMAGE_TAG}"
              docker logout "${DOCKER_REGISTRY}"
              echo "Docker image pushed: ${IMAGE_TAG}"
            '''
          }
        }
      }
    }

  } // stages

  post {
    success {
      echo "Pipeline finished SUCCESS"
    }
    failure {
      echo "Pipeline finished FAILURE"
    }
    always {
      // keep repo for debugging
      echo "Workspace listing (top):"
      sh 'ls -la || true'
    }
  }
}
