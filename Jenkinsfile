pipeline {
  agent any
  environment {
    IMAGE_NAME = 'devsecops-demo'
    SONAR_TOKEN = credentials('sonar-token') // create this in Jenkins credentials
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Test') {
      steps {
        sh 'echo "Run unit tests (if any)"; true'
        // If Node app: sh 'npm install && npm test'
        // If Java: sh 'mvn clean test'
      }
    }

    stage('SAST - SonarQube') {
      steps {
        withSonarQubeEnv('SonarQube') {
          // Example for Node project using sonar-scanner in container (adjust if Java)
          sh '''
            docker run --rm \
              -v "${PWD}:/usr/src" \
              -e SONAR_HOST_URL="http://host.docker.internal:9000" \
              -e SONAR_LOGIN=${SONAR_TOKEN} \
              sonarsource/sonar-scanner-cli \
              -Dsonar.projectKey=devsecops-demo -Dsonar.sources=/usr/src
          '''
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
        sh "docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest || true"
      }
    }

    stage('Container Security Scan') {
      steps {
        sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --format json --output trivy-report.json ${IMAGE_NAME}:${BUILD_NUMBER} || true"
        archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
      }
    }

    stage('Security Gate Check') {
      steps {
        script {
          def trivy = readJSON file: 'trivy-report.json'
          def critical=0
          trivy.Results.each { r ->
            (r.Vulnerabilities ?: []).each { v ->
                if (v.Severity == 'CRITICAL') critical++
            }
          }
          if (critical > 0) {
            error "Security Gate Failed: ${critical} CRITICAL vulnerabilities found"
          } else {
            echo "Security Gate Passed: No CRITICAL vulnerabilities"
          }
        }
      }
    }

    stage('Deploy to Staging') {
      steps {
        sh '''
          kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f -
          kubectl apply -f k8s/staging/ -n staging
          kubectl set image deployment/app app=${IMAGE_NAME}:${BUILD_NUMBER} -n staging || true
          kubectl rollout status deployment/app -n staging
        '''
      }
    }

    stage('DAST - ZAP') {
      steps {
        sh "docker run --rm owasp/zap2docker-stable zap-baseline.py -t http://$(minikube ip):30000 -J zap-report.json || true"
        archiveArtifacts artifacts: 'zap-report.json', allowEmptyArchive: true
      }
    }
  }

  post {
    always {
      sh 'python3 scripts/generate-simple-report.py || true'
      archiveArtifacts artifacts: 'security-report.html', allowEmptyArchive: true
    }
  }
}
