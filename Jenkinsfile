pipeline {
  agent any
  environment {
    MAVEN_HOME = tool 'Maven3'
    SONAR_TOKEN = credentials('sonar-token')
  }
  stages {
    stage('Checkout') {
      steps {
        // HTTPS clone using Jenkins Username+PAT credential (github-token)
        git branch: 'main',
            credentialsId: 'github-token',
            url: 'https://github.com/kanishkdw/mita-ci-cd-java.git'
      }
    }
    stage('Build') {
      steps {
        sh "${MAVEN_HOME}/bin/mvn -B clean package -DskipTests"
      }
    }
    stage('Unit Tests') {
      steps {
        sh "${MAVEN_HOME}/bin/mvn test"
        junit '**/target/surefire-reports/*.xml'
      }
    }
    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('SonarQube') {
          sh "${MAVEN_HOME}/bin/mvn sonar:sonar -Dsonar.login=${SONAR_TOKEN}"
        }
      }
    }
    stage('Docker Build & Run') {
      steps {
        sh "docker rm -f mita-app || true"
        sh "docker build -t mita-app:latest ."
        sh "docker run -d --rm --name mita-app -p 8080:8080 mita-app:latest"
      }
    }
    stage('Smoke Test') {
      steps {
        sh "curl -f http://localhost:8080/health"
      }
    }
  }
  post {
    always {
      archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }
  }
}
