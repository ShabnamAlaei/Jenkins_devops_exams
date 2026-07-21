pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "shabbyalaei"

        MOVIE_IMAGE = "shabbyalaei/movie-service"
        CAST_IMAGE  = "shabbyalaei/cast-service"

        IMAGE_TAG = "v.${BUILD_NUMBER}"

        DOCKER_PASS = credentials("DOCKER_HUB_PASS")
        KUBECONFIG_SECRET = credentials("KUBECONFIG_FILE")
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate') {
            steps {
                sh '''
                    docker compose config > /dev/null
                    helm lint charts
                    kubectl apply \
                      --dry-run=client \
                      -f k8s/databases.yaml \
                      --namespace dev
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                      -t $MOVIE_IMAGE:$IMAGE_TAG \
                      movie-service

                    docker build \
                      -t $CAST_IMAGE:$IMAGE_TAG \
                      cast-service
                '''
            }
        }

        stage('Docker Compose Test') {
            steps {
                sh '''
                    docker compose down --remove-orphans || true
                    docker compose up -d

                    sleep 15

                    curl -fsS \
                      http://localhost:8080/api/v1/movies/docs \
                      > /dev/null

                    curl -fsS \
                      http://localhost:8080/api/v1/casts/docs \
                      > /dev/null
                '''
            }
        }

        stage('Docker Push') {
            steps {
                sh '''
                    echo "$DOCKER_PASS_PSW" |
                    docker login \
                        -u "$DOCKER_PASS_USR" \
                        --password-stdin

                    docker push $MOVIE_IMAGE:$IMAGE_TAG
                    docker push $CAST_IMAGE:$IMAGE_TAG
                '''
            }
        }

        stage('Prepare Kubernetes') {
            steps {
                sh '''
                    rm -rf .kube
                    mkdir -p .kube

                    cp "$KUBECONFIG_SECRET" .kube/config
                    chmod 600 .kube/config

                    for ns in dev qa staging prod; do
                      kubectl create namespace "$ns" \
                        --dry-run=client \
                        -o yaml |
                      kubectl apply \
                        --kubeconfig .kube/config \
                        -f -
                    done
                '''
            }
        }

        stage('Deploy Dev') {
            steps {
                deployEnvironment(
                    "dev",
                    "30101",
                    "30102"
                )
            }
        }

        stage('Deploy QA') {
            steps {
                deployEnvironment(
                    "qa",
                    "30201",
                    "30202"
                )
            }
        }

        stage('Deploy Staging') {
            steps {
                deployEnvironment(
                    "staging",
                    "30301",
                    "30302"
                )
            }
        }

        stage('Deploy Production') {
            when {
                branch 'master'
            }

            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input(
                        message: 'Deploy to production?',
                        ok: 'Yes'
                    )
                }

                deployEnvironment(
                    "prod",
                    "30401",
                    "30402"
                )
            }
        }
    }

    post {
        always {
            sh '''
                docker compose down \
                  --remove-orphans || true
            '''
        }

        success {
            echo "Pipeline completed successfully."
        }

        failure {
            echo "Pipeline failed. Check the console output."
        }
    }
}

def deployEnvironment(
    String namespace,
    String movieNodePort,
    String castNodePort
) {
    sh """
        kubectl apply \
          --kubeconfig .kube/config \
          --namespace ${namespace} \
          -f k8s/databases.yaml

        kubectl rollout status \
          deployment/movie-db \
          --kubeconfig .kube/config \
          --namespace ${namespace} \
          --timeout=120s

        kubectl rollout status \
          deployment/cast-db \
          --kubeconfig .kube/config \
          --namespace ${namespace} \
          --timeout=120s

        cp values-dev-movie.yaml \
          values-${namespace}-movie.yaml

        cp values-dev-cast.yaml \
          values-${namespace}-cast.yaml

        sed -i \
          's|tag: "v1"|tag: "'"\$IMAGE_TAG"'"|' \
          values-${namespace}-movie.yaml

        sed -i \
          's|tag: "v1"|tag: "'"\$IMAGE_TAG"'"|' \
          values-${namespace}-cast.yaml

        sed -i \
          's/nodePort: [0-9]*/nodePort: ${movieNodePort}/' \
          values-${namespace}-movie.yaml

        sed -i \
          's/nodePort: [0-9]*/nodePort: ${castNodePort}/' \
          values-${namespace}-cast.yaml

        helm upgrade --install movie-service charts \
          --values values-${namespace}-movie.yaml \
          --namespace ${namespace} \
          --kubeconfig .kube/config

        helm upgrade --install cast-service charts \
          --values values-${namespace}-cast.yaml \
          --namespace ${namespace} \
          --kubeconfig .kube/config

        kubectl rollout status \
          deployment/movie-service-fastapiapp \
          --kubeconfig .kube/config \
          --namespace ${namespace} \
          --timeout=120s

        kubectl rollout status \
          deployment/cast-service-fastapiapp \
          --kubeconfig .kube/config \
          --namespace ${namespace} \
          --timeout=120s

        kubectl get pods,services \
          --kubeconfig .kube/config \
          --namespace ${namespace}
    """
}