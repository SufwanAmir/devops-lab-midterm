pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        IMAGE = 'sufwanamir/devops-midterm-api:1.0'
        CONTAINER = 'midterm-api'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.BUILD_SHA = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Train Model') {
            steps {
                sh '''
                    docker run --rm \
                      -v "$WORKSPACE:/work" \
                      -w /work \
                      -e GIT_COMMIT="$BUILD_SHA" \
                      python:3.12-slim \
                      bash -c "pip install --no-cache-dir -r requirements.txt && python train.py"

                    chown -R "$(id -u):$(id -g)" artifacts
                '''
            }
        }

        stage('Validate Metrics') {
            steps {
                sh '''
                    python3 -c "
import json
m=json.load(open('artifacts/metrics.json'))
assert m['student_id'] == 'FA23-BDS-063'
assert m['training_rows'] >= 100
assert 0 <= m['accuracy'] <= 1
assert m['git_sha'] == '$BUILD_SHA'
print('Metrics validation passed:', m)
"
                '''
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build -t $IMAGE .'
            }
        }

        stage('Test API') {
            steps {
                sh '''
                    docker rm -f midterm-api-test 2>/dev/null || true
                    docker run -d --name midterm-api-test \
                      -p 18000:8000 "$IMAGE"

                    trap 'docker rm -f midterm-api-test >/dev/null 2>&1 || true' EXIT

                    for i in $(seq 1 30); do
                        if curl -fsS http://localhost:18000/metrics -o /tmp/midterm-test.json; then
                            break
                        fi
                        sleep 2
                    done

                    python3 -c "
import json
m=json.load(open('/tmp/midterm-test.json'))
assert m['student_id'] == 'FA23-BDS-063'
assert m['git_sha'] == '$BUILD_SHA'
print('API test passed:', m)
"
                '''
            }
        }

        stage('Publish Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_TOKEN'
                )]) {
                    sh '''
                        echo "$DOCKER_TOKEN" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push "$IMAGE"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sh '''
                    docker rm -f "$CONTAINER" 2>/dev/null || true
                    docker run -d \
                      --name "$CONTAINER" \
                      -p 8000:8000 \
                      --restart unless-stopped \
                      "$IMAGE"

                    sleep 5
                    curl -fsS http://localhost:8000/metrics
                '''
            }
        }
    }


    post {
        success {
            script {
                def pushTriggered = currentBuild.getBuildCauses().any { cause ->
                    def description = cause['shortDescription'] ?: ''
                    description.toLowerCase().contains('github push by')
                }

                if (pushTriggered) {
                    def recipient = sh(
                        script: 'git log -1 --format=%ae',
                        returnStdout: true
                    ).trim()

                    if (recipient && recipient ==~ /[^\s@]+@[^\s@]+\.[^\s@]+/) {
                        emailext(
                            to: recipient,
                            from: 'sufwanamir99@gmail.com',
                            subject: 'CSC418 Midterm - FA23-BDS-063 - Pipeline SUCCESS',
                            body: """
The automated ML pipeline completed successfully.

Student ID: FA23-BDS-063
Build Number: ${BUILD_NUMBER}
Git Commit: ${BUILD_SHA}
Docker Image: sufwanamir/devops-midterm-api:1.0

API Metrics: http://13.60.111.25:8000/metrics
Jenkins Build: ${BUILD_URL}
"""
                        )
                    } else {
                        echo "No valid commit author email found. Notification skipped."
                    }
                }
            }
        }
    }
}
