pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 20, unit: 'MINUTES')
    }

    environment {
        APP_IMAGE = 'springboot-demo:latest'
        K8S_NAMESPACE = 'default'
        DEPLOYMENT_NAME = 'springboot-demo'
        SERVICE_NAME = 'springboot-demo'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify tools and cluster') {
            steps {
                sh '''
                    set -eu
                    git --version
                    java -version
                    mvn -version
                    docker version
                    kubectl config current-context
                    kubectl get nodes
                '''
            }
        }

        stage('Test and Package') {
            steps {
                sh 'mvn -B clean verify'
                junit 'target/surefire-reports/*.xml'
                archiveArtifacts artifacts: 'target/springboot-demo.jar', fingerprint: true
            }
        }

        stage('Build Docker image') {
            steps {
                sh '''
                    set -eu
                    docker build -t "$APP_IMAGE" .
                    docker image inspect "$APP_IMAGE"
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    set -eu
                    kubectl apply -f k8s/deployment.yaml
                    kubectl rollout status deployment/"$DEPLOYMENT_NAME" --timeout=180s
                    kubectl get pods -o wide
                    kubectl get service "$SERVICE_NAME"
                '''
            }
        }

        stage('Smoke test') {
            steps {
                sh '''
                    set -eu
                    kubectl port-forward service/"$SERVICE_NAME" 18080:80 >/tmp/springboot-port-forward.log 2>&1 &
                    PF_PID=$!
                    trap 'kill "$PF_PID" 2>/dev/null || true' EXIT
                    attempt=0
                    until curl --fail --silent http://127.0.0.1:18080/api/hello >/tmp/springboot-response.json; do
                      attempt=$((attempt + 1))
                      if [ "$attempt" -ge 20 ]; then
                        cat /tmp/springboot-port-forward.log || true
                        kubectl describe deployment "$DEPLOYMENT_NAME" || true
                        kubectl logs deployment/"$DEPLOYMENT_NAME" --tail=100 || true
                        exit 1
                      fi
                      sleep 2
                    done
                    cat /tmp/springboot-response.json
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline succeeded: Spring Boot has been deployed to Kubernetes.'
        }
        failure {
            echo 'Pipeline failed. Review the stage logs and Kubernetes pod events.'
        }
    }
}
