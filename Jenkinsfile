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
		// Étape 1 : Construire le Backend
		// ======================================================
		stage('Build Backend') {
			agent {
				docker {
					image 'maven:3.8.5-amazoncorretto-17'
					args '-v $HOME/.m2:/root/.m2'
				}
			}
			steps {
				git 'https://github.com/hamzaabidabid/job-tracker-backend.git'
				sh 'mvn clean package -DskipTests'
				stash name: 'backend-app', includes: 'target/job_tracker-0.0.1-SNAPSHOT.jar, Dockerfile, k8s/'
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
				stash name: 'frontend-app', includes: 'dist/**, Dockerfile, nginx.conf, k8s/'
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
			// On n'a pas besoin de 'dir' ou 'unstash' ici
			steps {
				script {
					def imageTag = "v1.${BUILD_NUMBER}"
					withCredentials([string(credentialsId: KUBERNETES_TOKEN_CREDENTIAL_ID, variable: 'KUBERNETES_TOKEN')]) {
						sh '''
                            kubectl config set-cluster minikube --server=${KUBERNETES_SERVER_URL} --insecure-skip-tls-verify=true
                            kubectl config set-credentials jenkins-agent --token=${KUBERNETES_TOKEN}
                            kubectl config set-context jenkins-context --cluster=minikube --user=jenkins-agent
                            kubectl config use-context jenkins-context

                            # --- CORRECTION ICI ---
                            # On applique tous les fichiers du dossier k8s qui a été cloné
                            # avec le Jenkinsfile au début du pipeline.
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