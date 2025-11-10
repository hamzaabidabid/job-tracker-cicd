pipeline {
	agent any

	environment {
		DOCKER_REGISTRY = 'abidhamza'
		BACKEND_IMAGE_NAME = "${DOCKER_REGISTRY}/job-tracker-backend"
		FRONTEND_IMAGE_NAME = "${DOCKER_REGISTRY}/job-tracker-frontend"
		KUBERNETES_SERVER_URL = 'https://192.168.49.2:8443'
		KUBERNETES_TOKEN_CREDENTIAL_ID = 'kubernetes-token'
	}

	stages {
		// ======================================================
		// Étape 1 : Construire le Backend et le Frontend en parallèle
		// ======================================================
		stage('Build Backend & Frontend') {
			parallel {
				// --- Branche pour le Backend ---
				stage('Build Backend') {
					agent any
					steps {
						echo "--- Building Backend ---"
						// On clone le dépôt du backend dans un sous-dossier
						dir('backend') {
							git 'https://github.com/hamzaabidabid/job-tracker-backend.git'

							// On utilise un agent Docker pour Maven
							docker.image('maven:3.8.5-openjdk-17').inside('-v $HOME/.m2:/root/.m2') {
								sh 'mvn clean package -DskipTests'
							}

							// On construit l'image Docker du backend
							script {
								def imageTag = "v1.${BUILD_NUMBER}"
								docker.build("${BACKEND_IMAGE_NAME}:${imageTag}", '.')
							}
						}
					}
				}

				// --- Branche pour le Frontend ---
				stage('Build Frontend') {
					agent any
					steps {
						echo "--- Building Frontend ---"
						// On clone le dépôt du frontend dans un autre sous-dossier
						dir('frontend') {
							git 'https://github.com/hamzaabidabid/job-tracker-frontend.git'

							// On utilise un agent Docker pour Node
							docker.image('node:18-alpine').inside {
								sh 'npm install'
								sh 'npm run build'
							}

							// On construit l'image Docker du frontend
							script {
								def imageTag = "v1.${BUILD_NUMBER}"
								docker.build("${FRONTEND_IMAGE_NAME}:${imageTag}", '.')
							}
						}
					}
				}
			}
		}

		// ======================================================
		// Étape 2 : Pousser les images sur Docker Hub
		// ======================================================
		stage('Push Docker Images') {
			steps {
				script {
					def imageTag = "v1.${BUILD_NUMBER}"
					docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-credentials') {
						// On pousse les deux images
						docker.image("${BACKEND_IMAGE_NAME}:${imageTag}").push()
						docker.image("${FRONTEND_IMAGE_NAME}:${imageTag}").push()
					}
				}
			}
		}

		// ======================================================
		// Étape 3 : Déployer toute l'application sur Kubernetes
		// ======================================================
		stage('Deploy to Kubernetes') {
			steps {
				script {
					def imageTag = "v1.${BUILD_NUMBER}"
					withCredentials([string(credentialsId: KUBERNETES_TOKEN_CREDENTIAL_ID, variable: 'KUBERNETES_TOKEN')]) {
						sh '''
                            kubectl config set-cluster minikube --server=${KUBERNETES_SERVER_URL} --insecure-skip-tls-verify=true
                            kubectl config set-credentials jenkins-agent --token=${KUBERNETES_TOKEN}
                            kubectl config set-context jenkins-context --cluster=minikube --user=jenkins-agent
                            kubectl config use-context jenkins-context

                            # On applique tous les fichiers de configuration
                            kubectl apply -f k8s/

                            # On met à jour les images des deux déploiements
                            kubectl set image deployment/backend-deployment backend-app=${BACKEND_IMAGE_NAME}:${imageTag}
                            kubectl set image deployment/frontend-deployment frontend-app=${FRONTEND_IMAGE_NAME}:${imageTag}

                            # On attend que les deux déploiements soient terminés
                            kubectl rollout status deployment/backend-deployment
                            kubectl rollout status deployment/frontend-deployment
                        '''
					}
				}
			}
		}
	}
}