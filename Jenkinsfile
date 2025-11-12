pipeline {
	agent any

	environment {
		DOCKER_REGISTRY = 'abidhamza'
		BACKEND_IMAGE_NAME = "${DOCKER_REGISTRY}/job-tracker-backend"
		FRONTEND_IMAGE_NAME = "${DOCKER_REGISTRY}/job-tracker-frontend"
	}

	stages {
		// ... (Les étapes 'Build Backend', 'Build Frontend', 'Build and Push Images' ne changent pas) ...
		stage('Build Backend') {
			agent { docker { image 'maven:3.8.5-amazoncorretto-17' } }
			steps {
				git 'https://github.com/hamzaabidabid/job-tracker-backend.git'
				sh 'mvn clean package -DskipTests'
				stash name: 'backend-app', includes: 'target/job_tracker-0.0.1-SNAPSHOT.jar, Dockerfile, k8s/'
			}
		}
		stage('Build Frontend') {
			agent { docker { image 'node:18-alpine' } }
			steps {
				git 'https://github.com/hamzaabidabid/job-tracker-frontend.git'
				sh 'npm install'
				sh 'npm run build'
				stash name: 'frontend-app', includes: 'dist/**, Dockerfile, nginx.conf, k8s/'
			}
		}
		stage('Build and Push Images') {
			steps {
				script {
					def imageTag = "v1.${BUILD_NUMBER}"
					docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-credentials') {
						dir('backend-workspace') {
							unstash 'backend-app'
							docker.build("${BACKEND_IMAGE_NAME}:${imageTag}", '.').push()
						}
						dir('frontend-workspace') {
							unstash 'frontend-app'
							docker.build("${FRONTEND_IMAGE_NAME}:${imageTag}", '.').push()
						}
					}
				}
			}
		}

		// ======================================================
		// Étape 4 : Déployer sur Kubernetes (NOUVELLE VERSION PROPRE)
		// ======================================================
		stage('Deploy to Kubernetes') {
			steps {
				// On récupère le dossier k8s qui contient nos manifestes
				unstash 'backend-app'

				script {
					def imageTag = "v1.${BUILD_NUMBER}"

					// On utilise le plugin Kubernetes CLI. C'est la méthode recommandée.
					// Il lit le credential 'kubeconfig-credentials' (qui doit être de type Secret File ou Secret Text).
					withKubeConfig([credentialsId: 'kubeconfig-credentials']) {

						echo "--- Applying all manifests ---"
						sh "kubectl apply -f k8s/"

						echo "--- Waiting for Database to be ready ---"
						sh "kubectl rollout status statefulset/postgres-db --timeout=5m"

						echo "--- Updating images ---"
						// Les variables fonctionnent parfaitement dans des commandes 'sh' simples.
						sh "kubectl set image deployment/backend-deployment backend-app=${BACKEND_IMAGE_NAME}:${imageTag}"
						sh "kubectl set image deployment/frontend-deployment frontend-app=${FRONTEND_IMAGE_NAME}:${imageTag}"

						echo "--- Waiting for application deployments to complete ---"
						sh "kubectl rollout status deployment/backend-deployment --timeout=5m"
						sh "kubectl rollout status deployment/frontend-deployment --timeout=5m"
					}
				}
			}
		}
	}
}