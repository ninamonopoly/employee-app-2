pipeline {
    agent any

    environment {
        AWS_REGION      = 'us-east-1'
        ECR_REGISTRY    = '075120018043.dkr.ecr.us-east-1.amazonaws.com'
        BACKEND_REPO    = "${ECR_REGISTRY}/employee-backend"
        FRONTEND_REPO   = "${ECR_REGISTRY}/employee-frontend"
    }

    stages {

        // ── Stage 1: Test ─────────────────────────────────────────────────
        stage('Test') {
            steps {
                sh '''
                    python3 -m venv venv
                    venv/bin/pip install -r backend/requirements.txt
                    venv/bin/pip install pytest
                    DATABASE_URL=sqlite:///test.db \
                    AWS_REGION=us-east-1 \
                    S3_BUCKET=test-bucket \
                    venv/bin/pytest backend/ -v --junitxml=backend/test-results/results.xml
                '''
            }
            post {
                always {
                    junit 'backend/test-results/results.xml'
                }
            }
        }

        // ── Stage 2: Build & Push ─────────────────────────────────────────
        stage('Build & Push') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials'
                ]]) {
                    sh '''
                        TIMESTAMP=$(date +\'%Y%m%d-%H%M%S\')
                        BACKEND_TAG="be-dev-${TIMESTAMP}"
                        FRONTEND_TAG="fe-dev-${TIMESTAMP}"

                        echo $BACKEND_TAG > /tmp/backend_tag
                        echo $FRONTEND_TAG > /tmp/frontend_tag

                        aws ecr get-login-password --region $AWS_REGION | \
                            docker login --username AWS --password-stdin $ECR_REGISTRY

                        docker build -t $BACKEND_REPO:$BACKEND_TAG backend/
                        docker push $BACKEND_REPO:$BACKEND_TAG

                        docker build -t $FRONTEND_REPO:$FRONTEND_TAG frontend/
                        docker push $FRONTEND_REPO:$FRONTEND_TAG
                    '''
                }
            }
        }

        // ── Stage 3: Update Helm values ───────────────────────────────────
        stage('Update Image Tags') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-credentials',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh '''
                        BACKEND_TAG=$(cat /tmp/backend_tag)
                        FRONTEND_TAG=$(cat /tmp/frontend_tag)

                        sed -i "0,/tag: .*/s/tag: .*/tag: ${BACKEND_TAG}/" kubernetes/helm/values.yaml
                        sed -i "0,/tag: ${BACKEND_TAG}/!s/tag: .*/tag: ${FRONTEND_TAG}/" kubernetes/helm/values.yaml

                        git config user.name "jenkins"
                        git config user.email "jenkins@landmark.dev"
                        git pull --rebase https://${GIT_USER}:${GIT_TOKEN}@github.com/ninamonopoly/employee-app-2.git main
                        git add kubernetes/helm/values.yaml
                        git diff --cached --quiet && echo "No changes to commit" && exit 0
                        git commit -m "ci: update tags - backend:${BACKEND_TAG} frontend:${FRONTEND_TAG}"
                        git push https://${GIT_USER}:${GIT_TOKEN}@github.com/ninamonopoly/employee-app-2.git HEAD:main
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline succeeded'
        }
        failure {
            echo 'Pipeline failed — check logs above'
        }
    }
}