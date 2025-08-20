pipeline {
  agent { label 'Jenkins-Agent' }

  tools {
    jdk 'Java17'
    maven 'Maven3'
  }

  environment {
    APP_NAME   = "register-app-pipeline"
    RELEASE    = "1.0.0"
    IMAGE_TAG  = "${RELEASE}-${BUILD_NUMBER}"
    // Don't put secrets here. We will use withCredentials.
  }

  stages {
    stage('Cleanup Workspace') {
      steps { cleanWs() }
    }

    stage('Checkout from SCM') {
      steps {
        git branch: 'main', credentialsId: 'github',
            url: 'https://github.com/Nagaraju-209/register-app'
      }
    }

    stage('Build & Package') {
      steps { sh 'mvn -B -DskipTests clean package' }
    }

    stage('Run Unit Tests') {
      steps {
        sh 'mvn -B test'
        junit 'server/target/surefire-reports/*.xml'
      }
    }

    stage('SonarQube Analysis') {
      steps {
        script {
          withSonarQubeEnv('MySonar') {
            sh 'mvn -B verify sonar:sonar'
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        script {
          timeout(time: 5, unit: 'MINUTES') {
            def qg = waitForQualityGate()
            if (qg.status != 'OK') {
              error "Quality Gate failed: ${qg.status}"
            }
          }
        }
      }
    }

    stage('Build & Push Docker Image') {
      steps {
        script {
          withCredentials([usernamePassword(credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh '''
              echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
              docker build -t ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG} .
              docker tag ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG} ${DOCKER_USER}/${APP_NAME}:latest
              docker push ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG}
              docker push ${DOCKER_USER}/${APP_NAME}:latest
            '''
          }
        }
      }
    }

    stage('Trivy Scan') {
      steps {
        sh '''
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy image --no-progress --scanners vuln \
            --exit-code 0 --severity HIGH,CRITICAL --format table \
            $(echo ${APP_NAME} | sed "s@^@${DOCKER_USER}/@"):latest || true
        '''
      }
    }

    stage('Cleanup Artifacts') {
      steps {
        sh '''
          docker rmi ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG} || true
          docker rmi ${DOCKER_USER}/${APP_NAME}:latest || true
        '''
      }
    }

    stage('Trigger CD Pipeline') {
      steps {
        script {
          withCredentials([usernamePassword(credentialsId: 'jenkins-trigger',
                        usernameVariable: 'TRIGGER_USER', passwordVariable: 'TRIGGER_TOKEN')]) {
            sh '''
              # Get Jenkins crumb
              CRUMB_JSON=$(curl -s -u "$TRIGGER_USER:$TRIGGER_TOKEN" "http://<EC2_PUBLIC_IP>:8080/crumbIssuer/api/json")
              CRUMB=$(echo "$CRUMB_JSON" | python3 -c "import sys, json; print(json.load(sys.stdin)['crumb'])")
              # Trigger downstream job with crumb & params
              curl -s -u "$TRIGGER_USER:$TRIGGER_TOKEN" -H "Jenkins-Crumb:$CRUMB" \
                -X POST "http://<EC2_PUBLIC_IP>:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token" \
                --data "IMAGE_TAG=${IMAGE_TAG}"
            '''
          }
        }
      }
    }
  }

  post {
    success {
      emailext(
        subject: "${env.JOB_NAME} #${env.BUILD_NUMBER} SUCCESS",
        body: '${SCRIPT, template="groovy-html.template"}',
        mimeType: 'text/html',
        to: 'relangimaya5@gmail.com'
      )
    }
    failure {
      emailext(
        subject: "${env.JOB_NAME} #${env.BUILD_NUMBER} FAILED",
        body: '${SCRIPT, template="groovy-html.template"}',
        mimeType: 'text/html',
        to: 'relangimaya5@gmail.com'
      )
    }
  }
}
