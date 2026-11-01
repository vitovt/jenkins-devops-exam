def deployEnv(String ns) {
    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
        withEnv(["DEPLOY_NS=${ns}"]) {
            sh '''
                set -eu
                helm upgrade --install cinema ./charts \
                  --namespace "$DEPLOY_NS" \
                  --set-string movie.image.repository="$MOVIE_IMAGE" \
                  --set-string movie.image.tag="$IMAGE_TAG" \
                  --set-string cast.image.repository="$CAST_IMAGE" \
                  --set-string cast.image.tag="$IMAGE_TAG" \
                  --wait --atomic --timeout 5m
                helm test cinema --namespace "$DEPLOY_NS" --logs --timeout 3m
                kubectl get pods --namespace "$DEPLOY_NS"
            '''
        }
    }
}

pipeline {
    agent any
    options {
        timestamps()
        disableConcurrentBuilds()
    }
    environment {
        MOVIE_IMAGE = 'vvitovt/jenkins-exam-movie'
        CAST_IMAGE  = 'vvitovt/jenkins-exam-cast'
    }
    stages {
        stage('Init') {
            steps {
                script {
                    env.IMAGE_TAG = "${env.GIT_COMMIT.take(8)}-${env.BUILD_NUMBER}"
                }
                sh 'echo "Branch: $BRANCH_NAME, tag: $IMAGE_TAG"'
            }
        }
        stage('Validate') {
            steps {
                sh '''
                    set -eu
                    docker compose config --quiet
                    helm lint ./charts
                    helm template cinema ./charts > /dev/null
                '''
            }
        }
        stage('Build Images') {
            steps {
                sh 'docker compose -p cinema-ci build'
            }
        }
        stage('Compose Test') {
            steps {
                sh '''
                    set -eu
                    docker compose -p cinema-ci up -d --no-build
                    curl -fsS --retry 30 --retry-delay 3 --retry-connrefused -o /dev/null http://localhost:8080/api/v1/movies/docs
                    curl -fsS --retry 30 --retry-delay 3 --retry-connrefused -o /dev/null http://localhost:8080/api/v1/casts/docs
                '''
            }
            post {
                always {
                    sh '''
                        docker compose -p cinema-ci logs --no-color > compose.log 2>&1 || true
                        docker compose -p cinema-ci down -v --remove-orphans || true
                    '''
                    archiveArtifacts artifacts: 'compose.log', allowEmptyArchive: true
                }
            }
        }
        stage('Push Images') {
            when { branch 'master' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub',
                                                  usernameVariable: 'DH_LOGIN',
                                                  passwordVariable: 'DH_TOKEN')]) {
                    sh '''
                        set -eu
                        printf '%s' "$DH_TOKEN" | docker login -u "$DH_LOGIN" --password-stdin
                        docker push "${MOVIE_IMAGE}:${IMAGE_TAG}"
                        docker push "${CAST_IMAGE}:${IMAGE_TAG}"
                        docker logout
                    '''
                }
            }
        }
        stage('Deploy DEV') {
            when { branch 'master' }
            steps { script { deployEnv('dev') } }
        }
        stage('Deploy QA') {
            when { branch 'master' }
            steps { script { deployEnv('qa') } }
        }
        stage('Deploy STAGING') {
            when { branch 'master' }
            steps { script { deployEnv('staging') } }
        }
        stage('Deploy PROD') {
            options { timeout(time: 24, unit: 'HOURS') }
            when {
                beforeInput true
                branch 'master'
            }
            input {
                message "Deploy ${IMAGE_TAG} to PRODUCTION?"
                ok 'Deploy'
            }
            steps { script { deployEnv('prod') } }
        }
    }
}
