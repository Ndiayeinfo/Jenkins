pipeline {
    // Chaque stage choisit son propre environnement Docker
    agent none

    environment {
        // Noms des dépôts Docker Hub pour les images backend et frontend
        BACKEND_IMAGE  = 'ndiayeinf/backend-fastapi-gtp'
        FRONTEND_IMAGE = 'ndiayeinf/frontend-react-gtp'
    }

    stages {
        stage('Backend - Build & Tests') {
            agent {
                docker {
                    image 'python:3.13-slim'
                    args '-u root'
                }
            }
            steps {
                sh '''
                    set -e
                    cd backend-fastapi-gtp

                    echo "Installation des dépendances Python..."
                    pip install --no-cache-dir -r requirements.txt

                    echo "Recherche et exécution des tests backend (pytest) si présents..."
                    if [ -f "pytest.ini" ] || ls tests test_*.py >/dev/null 2>&1; then
                      pytest
                    else
                      echo "Aucun test backend détecté, étape pytest ignorée."
                    fi
                '''
            }
        }

        stage('Frontend - Build & Tests') {
            agent {
                docker {
                    image 'node:20-alpine'
                    args '-u root'
                }
            }
            steps {
                sh '''
                    set -e
                    cd frontend-react-gtp

                    echo "Installation des dépendances npm..."
                    npm install

                    echo "Exécution des tests frontend si le script test existe..."
                    if npm run | grep -q " test"; then
                      npm test -- --watchAll=false
                    else
                      echo "Aucun script de test frontend défini, étape ignorée."
                    fi

                    echo "Build de l'application frontend..."
                    npm run build
                '''
            }
        }

        stage('Build & Push Docker Images') {
            agent {
                docker {
                    image 'docker:25.0.3'
                    args '-u root -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'DOCKER_HUB_PASSWORD', usernameVariable: 'DOCKER_HUB_USERNAME')]) {
                    sh '''
                        set -e
                        # Désactiver BuildKit pour éviter l erreur "driver not connecting"
                        export DOCKER_BUILDKIT=0

                        echo "Connexion à Docker Hub..."
                        echo "$DOCKER_HUB_PASSWORD" | docker login -u "$DOCKER_HUB_USERNAME" --password-stdin

                        echo "Build et push de l'image backend..."
                        docker build -t ${BACKEND_IMAGE}:v${BUILD_NUMBER} ./backend-fastapi-gtp
                        docker push ${BACKEND_IMAGE}:v${BUILD_NUMBER}

                        echo "Build et push de l'image frontend..."
                        docker build -t ${FRONTEND_IMAGE}:v${BUILD_NUMBER} ./frontend-react-gtp
                        docker push ${FRONTEND_IMAGE}:v${BUILD_NUMBER}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline exécuté avec succès."
            echo "Backend -> ${BACKEND_IMAGE}:v${BUILD_NUMBER}"
            echo "Frontend -> ${FRONTEND_IMAGE}:v${BUILD_NUMBER}"
        }
        failure {
            echo "Le pipeline a échoué, merci de vérifier les logs."
        }
    }
}

