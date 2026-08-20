pipeline {
    agent any

    environment {
        AWS_REGION      = 'us-east-1'
        ECR_REGISTRY    = '075120018043.dkr.ecr.us-east-1.amazonaws.com'
        BACKEND_REPO    = "${ECR_REGISTRY}/employee-backend"
        FRONTEND_REPO   = "${ECR_REGISTRY}/employee-frontend"
        TIMESTAMP       = sh(script: "date +'%Y%m%d-%H%M%S'", returnStdout: true).trim()
        BACKEND_TAG     = "be-dev-${TIMESTAMP}"
        FRONTEND_TAG    = "fe-dev-${TIMESTAMP}"
    }

    stages {

        // ── Stage 1: Test ─────────────────────────────────────────────────
        stage('Test') {
            steps {
                    sh '''
                    pip install -r backend/requirements.txt
                    cd backend
                    DATABASE_URL=sqlite:///test.db \
                    AWS_REGION=us-east-1 \
                    pytest -v
                '''
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
                sh '''
                    sed -i "0,/tag: .*/s/tag: .*/tag: ${BACKEND_TAG}/" kubernetes/helm/values.yaml
                    sed -i "0,/tag: ${BACKEND_TAG}/!s/tag: .*/tag: ${FRONTEND_TAG}/" kubernetes/helm/values.yaml
                '''
                withCredentials([usernamePassword(
                    credentialsId: 'github-credentials',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh '''
                        git config user.name "jenkins"
                        git config user.email "jenkins@landmark.dev"
                        git add kubernetes/helm/values.yaml
                        git commit -m "ci: update tags - backend:${BACKEND_TAG} frontend:${FRONTEND_TAG}" || true
                        git push https://${GIT_USER}:${GIT_TOKEN}@github.com/ninamonopoly/employee-app-2.git HEAD:main
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded — backend:${BACKEND_TAG} frontend:${FRONTEND_TAG}"
        }
        failure {
            echo "Pipeline failed — check logs above"
        }
    }
}
