// Pipeline CI : construit les 3 images, les tague et les pousse sur Docker Hub.
// Le deploiement est assure par Argo CD, pas par ce pipeline.
pipeline {

    agent any

    environment {
        DOCKERHUB_USER = 'diafa'
        IMAGE_TAG      = "1.${BUILD_NUMBER}"        // tag unique par build
        BACKEND_IMAGE  = "${DOCKERHUB_USER}/bijouterie-backend"
        FRONTEND_IMAGE = "${DOCKERHUB_USER}/bijouterie-frontend"
        MYSQL_IMAGE    = "${DOCKERHUB_USER}/bijouterie-mysql"
        GIT_REPO       = 'github.com/bayefall13/bijouterie-microservices.git'
    }

    stages {

        // Recupere le code depuis le depot Git configure dans le job
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // Construit une image par microservice, chacune depuis son dossier
        stage('Build') {
            steps {
                sh """
                    docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} ./backend
                    docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} ./frontend
                    docker build -t ${MYSQL_IMAGE}:${IMAGE_TAG} ./database
                """
            }
        }

        // Ajoute un second tag latest : 1.x sert aux manifests, latest au registre
        stage('Tag') {
            steps {
                sh """
                    docker tag ${BACKEND_IMAGE}:${IMAGE_TAG} ${BACKEND_IMAGE}:latest
                    docker tag ${FRONTEND_IMAGE}:${IMAGE_TAG} ${FRONTEND_IMAGE}:latest
                    docker tag ${MYSQL_IMAGE}:${IMAGE_TAG} ${MYSQL_IMAGE}:latest
                """
            }
        }

        // Publie les 6 references sur Docker Hub.
        // Guillemets simples + --password-stdin : le token n'apparait pas en clair.
        stage('Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                        docker push ${BACKEND_IMAGE}:latest
                        docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                        docker push ${FRONTEND_IMAGE}:latest
                        docker push ${MYSQL_IMAGE}:${IMAGE_TAG}
                        docker push ${MYSQL_IMAGE}:latest

                        docker logout
                    '''
                }
            }
        }

        // GitOps : on ecrit le nouveau tag dans k8s/ et on pousse.
        // Argo CD detecte le commit et deploie la nouvelle version.
        stage('Update manifests') {
            steps {
                sh """
                    sed -i -E "s#(image: ${DOCKERHUB_USER}/bijouterie-[a-z]+):[^[:space:]]+#\\1:${IMAGE_TAG}#" k8s/*-deployment.yaml
                """

                withCredentials([usernamePassword(
                    credentialsId: 'github-credentials',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh '''
                        git config user.email "jenkins@bijouterie.local"
                        git config user.name "Jenkins CI"
                        git add k8s/

                        # [skip ci] evite que ce commit relance un build en boucle
                        git diff --cached --quiet || \
                          git commit -m "ci: images ${IMAGE_TAG} (build #${BUILD_NUMBER}) [skip ci]"

                        set +x
                        git push "https://${GIT_USER}:${GIT_TOKEN}@${GIT_REPO}" HEAD:main
                        set -x
                    '''
                }
            }
        }
    }

    post {
        // Libere le disque de l'agent, les images restent sur Docker Hub
        always {
            sh """
                docker rmi ${BACKEND_IMAGE}:${IMAGE_TAG} ${BACKEND_IMAGE}:latest || true
                docker rmi ${FRONTEND_IMAGE}:${IMAGE_TAG} ${FRONTEND_IMAGE}:latest || true
                docker rmi ${MYSQL_IMAGE}:${IMAGE_TAG} ${MYSQL_IMAGE}:latest || true
            """
        }
    }
}
