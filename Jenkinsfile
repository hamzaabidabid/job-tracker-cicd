pipeline {
	agent any // On définit un agent de base, mais chaque étape de build le remplacera

	environment {
		DOCKER_REGISTRY = 'abidhamza'
		BACKEND_IMAGE_NAME = "${DOCKER_REGISTRY}/job-tracker-backend"
		FRONTEND_IMAGE_NAME = "${DOCKER_REGISTRY}/job-tracker-frontend"
	}

	stages {
		// ======================================================
		// Étape 1 : Construire le Backend
		// ======================================================
		stage('Build Backend') {
			// On définit un agent Docker spécifique pour CETTE étape
			agent {
				docker {
					image 'maven:3.8.5-amazoncorretto-17'
					args '-v $HOME/.m2:/root/.m2'
				}
			}
			steps {
				echo "--- Building Backend ---"
				// On clone le dépôt du backend
				git 'https://github.com/hamzaabidabid/job-tracker-backend.git'
				// On exécute le build Maven
				sh 'mvn clean package -DskipTests'
				// On archive les fichiers nécessaires pour la prochaine étape
				stash name: 'backend-app', includes: 'target/job_tracker-0.0.1-SNAPSHOT.jar, Dockerfile'
			}
		}

		// ======================================================
		// Étape 2 : Construire le Frontend
		// ======================================================
		stage('Build Frontend') {
			agent {
				docker {
					image 'node:18-alpine'
				}
			}
			steps {
				echo "--- Building Frontend ---"
				git 'https://github.com/hamzaabidabid/job-tracker-frontend.git'
				sh 'npm install'
				sh 'npm run build'
				stash name: 'frontend-app', includes: 'dist/**, Dockerfile, nginx.conf'
			}
		}

		// ======================================================
		// Étape 3 : Construire et Pousser les Images Docker
		// ======================================================
		stage('Build and Push Images') {
			steps {
				script {
					def imageTag = "v1.${BUILD_NUMBER}"
					docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-credentials') {
						// Build et push du backend
						dir('backend-workspace') {
							unstash 'backend-app'
							docker.build("${BACKEND_IMAGE_NAME}:${imageTag}", '.').push()
						}
						// Build et push du frontend
						dir('frontend-workspace') {
							unstash 'frontend-app'
							docker.build("${FRONTEND_IMAGE_NAME}:${imageTag}", '.').push()
						}
					}
				}
			}
		}

		// ======================================================
		// Étape 4 : Déployer sur Kubernetes
		// ======================================================
		stage('Deploy to Kubernetes') {
			steps {
				// On clone le dépôt qui contient les fichiers k8s
				git 'https://github.com/hamzaabidabid/job-tracker-cicd.git'

				script {
					def imageTag = "v1.${BUILD_NUMBER}"

					// On utilise le plugin Kubernetes CLI pour s'authentifier
					withKubeConfig([credentialsId: 'kubeconfig-credentials']) {

						echo "--- Applying all manifests ---"
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