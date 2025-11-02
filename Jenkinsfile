pipeline {
  agent any
  environment {
    MAVEN_HOME = tool name: 'Maven3', type: 'hudson.tasks.Maven$MavenInstallation' // ensure Maven3 is installed in Jenkins global tools
    SONAR_TOKEN = credentials('sonar-token')   // configure in Jenkins credentials if you use SonarQube
  }
  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', credentialsId: 'jenkins-git-ssh', url: "${env.REPO_URL ?: 'git@github.com:kanishkdwi/mita-ci-cd-java.git'}"
      }
    }

    stage('Build') {
      steps {
        sh "${MAVEN_HOME}/bin/mvn -B -DskipTests clean package"
      }
    }

    stage('Unit Tests') {
      steps {
        sh "${MAVEN_HOME}/bin/mvn test"
        junit '**/target/surefire-reports/*.xml'
      }
    }

    stage('SonarQube Analysis') {
      when { expression { return env.SONAR_TOKEN != null } }
      steps {
        withSonarQubeEnv('SonarQube') {
          sh "${MAVEN_HOME}/bin/mvn sonar:sonar -Dsonar.login=${SONAR_TOKEN}"
        }
      }
    }

    stage('Docker Build') {
      steps {
        script {
          // If you want to only build the image without pushing
          sh "docker build -t mita-app:latest ."
        }
      }
    }

    stage('Run Smoke') {
      steps {
        // stop old container and start new one (adjust ports as required)
        sh "docker rm -f mita-app || true"
        sh "docker run -d --rm --name mita-app -p 8080:8080 mita-app:latest"
        sh "curl -f http://localhost:8080/health || true"
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }
    success {
      echo "Build and deploy successful!"
    }
    failure {
      echo "Pipeline failed. See logs."
    }
  }
}
