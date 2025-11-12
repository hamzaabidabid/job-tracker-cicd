pipeline {
	agent any

	environment {
		DOCKER_REGISTRY = 'abidhamza'
		BACKEND_IMAGE_NAME = "${DOCKER_REGISTRY}/job-tracker-backend"
		FRONTEND_IMAGE_NAME = "${DOCKER_REGISTRY}/job-tracker-frontend"
	}

	stages {
		// ======================================================
		// Étape 1 : Build & Push Backend
		// ======================================================
		stage('Build & Push Backend') {
			steps {
				echo "--- Building & Pushing Backend ---"
				// On clone le dépôt dans un sous-dossier pour l'isoler
				dir('backend') {
					git 'https://github.com/hamzaabidabid/job-tracker-backend.git'

					// On utilise un agent Docker pour Maven
					docker.image('maven:3.8.5-amazoncorretto-17').inside('-v $HOME/.m2:/root/.m2') {
						sh 'mvn clean package -DskipTests'
					}

					// On construit et on pousse l'image
					script {
						def imageTag = "v1.${BUILD_NUMBER}"
						def dockerImage = docker.build("${BACKEND_IMAGE_NAME}:${imageTag}", '.')
						docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-credentials') {
							dockerImage.push()
						}
					}
				}
			}
		}

		// ======================================================
		// Étape 2 : Build & Push Frontend
		// ======================================================
		stage('Build & Push Frontend') {
			steps {
				echo "--- Building & Pushing Frontend ---"
				dir('frontend') {
					git 'https://github.com/hamzaabidabid/job-tracker-frontend.git'

					docker.image('node:18-alpine').inside {
						sh 'npm install'
						sh 'npm run build'
					}

					script {
						def imageTag = "v1.${BUILD_NUMBER}"
						def dockerImage = docker.build("${FRONTEND_IMAGE_NAME}:${imageTag}", '.')
						docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-credentials') {
							dockerImage.push()
						}
					}
				}
			}
		}

		// ======================================================
		// Étape 3 : Déployer sur Kubernetes
		// ======================================================
		stage('Deploy to Kubernetes') {
			steps {
				script {
					def imageTag = "v1.${BUILD_NUMBER}"
					withKubeConfig([credentialsId: 'kubeconfig-credentials']) {

						echo "--- Applying all manifests ---"
						// On clone ce dépôt (cicd) pour avoir les fichiers k8s
						git 'https://github.com/hamzaabidabid/job-tracker-cicd.git'
						sh "kubectl apply -f k8s/"

						echo "--- Waiting for Database ---"
						sh "kubectl rollout status statefulset/postgres-db --timeout=5m"

						echo "--- Updating images ---"
						sh "kubectl set image deployment/backend-deployment backend-app=${BACKEND_IMAGE_NAME}:${imageTag}"
						sh "kubectl set image deployment/frontend-deployment frontend-app=${FRONTEND_IMAGE_NAME}:${imageTag}"

						echo "--- Waiting for deployments ---"
						sh "kubectl rollout status deployment/backend-deployment --timeout=5m"
						sh "kubectl rollout status deployment/frontend-deployment --timeout=5m"
					}
				}
			}
		}
	}
}