pipeline {
    agent any

    environment {
        AWS_REGION      = 'ap-south-1'
        ECR_REPO        = "951066974776.dkr.ecr.${AWS_REGION}.amazonaws.com/cicd-demo-app"
        IMAGE_TAG       = "${env.GIT_COMMIT.take(7)}"
        MANIFESTS_REPO  = 'git@github.com:hrithik-k0/cicd-demo-manifests.git'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Lint') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r requirements.txt
                    flake8 src --max-line-length=100
                '''
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    pytest tests/ --junitxml=test-results.xml
                '''
            }
            post {
                always {
                    junit 'test-results.xml'
                }
            }
        }

        stage('Secret Scan') {
            steps {
                sh '''
                    docker run --rm -v $(pwd):/repo zricethezav/gitleaks:latest \
                        detect --source=/repo --no-git -v || true
                '''
            }
        }

        stage('Build Image') {
            steps {
                sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} -t ${ECR_REPO}:latest ."
            }
        }

        stage('Container Vulnerability Scan') {
            steps {
                sh '''
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                    aquasec/trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 \
                    ${ECR_REPO}:${IMAGE_TAG}
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REPO}
                    docker push ${ECR_REPO}:${IMAGE_TAG}
                    docker push ${ECR_REPO}:latest
                '''
            }
        }

        stage('Update GitOps Manifests') {
            steps {
                sshagent(credentials: ['github-ssh-key']) {
                    sh '''
                        rm -rf manifests-repo
                        git clone ${MANIFESTS_REPO} manifests-repo
                        cd manifests-repo
                        sed -i "s#image: .*#image: ${ECR_REPO}:${IMAGE_TAG}#" base/deployment.yaml
                        git config user.email "jenkins@ci.local"
                        git config user.name "Jenkins CI"
                        git commit -am "Deploy ${IMAGE_TAG}" || echo "No changes to commit"
                        git push origin main
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded — ArgoCD will pick up image tag ${IMAGE_TAG} shortly."
        }
        failure {
            echo "Pipeline failed. Check the stage logs above."
        }
        always {
            sh 'docker system prune -f || true'
        }
    }
}
