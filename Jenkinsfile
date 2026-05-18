pipeline {
    agent any

    environment {
        GCP_PROJECT_ID         = "project-613709cc-f85d-4208-ad6"
        GCP_REGION             = "us-central1"
        ARTIFACT_REGISTRY_REPO = "container-repository-gemini-mpd"
        CLOUD_RUN_SERVICE_NAME = "gemini-angular-app"
        ENVIRONMENT_NAME       = "dev"

        GIT_COMMIT_SHORT = "${env.GIT_COMMIT?.take(7) ?: 'latest'}"
        IMAGE_TAG        = "${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT_ID}/${ARTIFACT_REGISTRY_REPO}/${CLOUD_RUN_SERVICE_NAME}:${GIT_COMMIT_SHORT}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm

                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = "${env.GCP_REGION}-docker.pkg.dev/${env.GCP_PROJECT_ID}/${env.ARTIFACT_REGISTRY_REPO}/${env.CLOUD_RUN_SERVICE_NAME}:${env.GIT_COMMIT_SHORT}"
                }
            }
        }

        stage('GCP Auth') {
            steps {
                withCredentials([file(credentialsId: 'gcp-service-account', variable: 'GCP_KEY')]) {
                    sh """
                        gcloud auth activate-service-account --key-file=${GCP_KEY}
                        gcloud config set project ${GCP_PROJECT_ID}
                        gcloud auth configure-docker ${GCP_REGION}-docker.pkg.dev --quiet
                    """
                }
            }
        }

        stage('Build and Push Image') {
            steps {
                sh """
                    docker build -t ${IMAGE_TAG} .
                    docker push ${IMAGE_TAG}
                """
            }
        }

        stage('Deploy to Cloud Run') {
            steps {
                withCredentials([file(credentialsId: 'gcp-service-account', variable: 'GCP_KEY')]) {
                    sh """
                        gcloud auth activate-service-account --key-file=${GCP_KEY}
                        gcloud config set project ${GCP_PROJECT_ID}

                        gcloud run deploy ${CLOUD_RUN_SERVICE_NAME}-${ENVIRONMENT_NAME} \
                            --image ${IMAGE_TAG} \
                            --region ${GCP_REGION} \
                            --platform managed \
                            --allow-unauthenticated \
                            --set-env-vars="ENVIRONMENT_NAME=${ENVIRONMENT_NAME}"
                    """
                }
            }
        }
    }

    post {
        always {
            sh """
                docker rmi ${IMAGE_TAG} || true
            """
        }
    }
}
