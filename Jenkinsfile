pipeline {
    agent { label 'built-in' }

    environment {
        PATH = "/opt/homebrew/bin:/usr/local/bin:${env.PATH}"
        REGISTRY_HOST = 'crpi-uzkholcdutt858o7.cn-guangzhou.personal.cr.aliyuncs.com'
        IMAGE_REPOSITORY = 'crpi-uzkholcdutt858o7.cn-guangzhou.personal.cr.aliyuncs.com/maxfan/kubernetes-cd-demo'
        IMAGE_PLATFORM = 'linux/arm64'
        KUBE_CONTEXT = 'kind-k8s-chapter1'
        KUBE_NAMESPACE = 'training'
        DEPLOYMENT_NAME = 'cd-demo-deployment'
        CONTAINER_NAME = 'cd-demo'
    }

    options {
        timestamps()
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        timeout(time: 20, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Validate') {
            steps {
                sh '''
                    set -eu
                    for file in app/index.html app/Dockerfile k8s/namespace.yaml \
                        k8s/deployment.yaml k8s/service.yaml k8s/ingress.yaml; do
                        test -s "$file"
                    done
                    grep -q 'Application version:' app/index.html
                    git --version
                    docker version
                    docker buildx version
                    kubectl version --client
                '''
            }
        }

        stage('Build and Push Image') {
            steps {
                sh '''
                    set -eu
                    docker buildx build --platform "$IMAGE_PLATFORM" \
                        --provenance=false --sbom=false --load \
                        -t "$IMAGE_REPOSITORY:$IMAGE_TAG" ./app
                '''
                withCredentials([
                    usernamePassword(
                        credentialsId: 'container-registry-credentials',
                        usernameVariable: 'REGISTRY_USERNAME',
                        passwordVariable: 'REGISTRY_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x
                        set -eu
                        # Preserve the Docker Desktop endpoint before isolating login data.
                        export DOCKER_HOST="${DOCKER_HOST:-$(docker context inspect --format '{{.Endpoints.docker.Host}}')}"
                        unset DOCKER_CONTEXT
                        DOCKER_CONFIG=$(mktemp -d)
                        export DOCKER_CONFIG
                        trap 'docker logout "$REGISTRY_HOST" >/dev/null 2>&1 || true; rm -rf "$DOCKER_CONFIG"' EXIT
                        printf '%s' "$REGISTRY_PASSWORD" | docker login "$REGISTRY_HOST" \
                            --username "$REGISTRY_USERNAME" --password-stdin
                        docker push --platform "$IMAGE_PLATFORM" "$IMAGE_REPOSITORY:$IMAGE_TAG"
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig-training', variable: 'KUBECONFIG_FILE')
                ]) {
                    sh '''
                        set +x
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"
                        kubectl --context="$KUBE_CONTEXT" apply -f k8s/namespace.yaml
                        kubectl --context="$KUBE_CONTEXT" -n "$KUBE_NAMESPACE" \
                            get secret cd-demo-acr-pull -o name

                        deployment_file=$(mktemp)
                        trap 'rm -f "$deployment_file"' EXIT
                        kubectl set image --local -f k8s/deployment.yaml \
                            "$CONTAINER_NAME=$IMAGE_REPOSITORY:$IMAGE_TAG" \
                            -o yaml > "$deployment_file"
                        kubectl --context="$KUBE_CONTEXT" apply \
                            -f k8s/service.yaml -f k8s/ingress.yaml -f "$deployment_file"
                        kubectl --context="$KUBE_CONTEXT" -n "$KUBE_NAMESPACE" rollout status \
                            deployment/"$DEPLOYMENT_NAME" --timeout=180s
                    '''
                }
            }
        }

        stage('Verify') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig-training', variable: 'KUBECONFIG_FILE')
                ]) {
                    sh '''
                        set +x
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"
                        actual_image=$(kubectl --context="$KUBE_CONTEXT" -n "$KUBE_NAMESPACE" \
                            get deployment "$DEPLOYMENT_NAME" \
                            -o "jsonpath={.spec.template.spec.containers[?(@.name=='$CONTAINER_NAME')].image}")
                        test "$actual_image" = "$IMAGE_REPOSITORY:$IMAGE_TAG"
                        kubectl --context="$KUBE_CONTEXT" -n "$KUBE_NAMESPACE" \
                            get deployments,pods,services,ingresses -l app=cd-demo -o wide
                        kubectl --context="$KUBE_CONTEXT" -n "$KUBE_NAMESPACE" \
                            get endpointslices -l kubernetes.io/service-name=cd-demo-service
                        kubectl --context="$KUBE_CONTEXT" -n "$KUBE_NAMESPACE" \
                            rollout history deployment/"$DEPLOYMENT_NAME"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Deployment completed successfully: ${env.IMAGE_REPOSITORY}:${env.IMAGE_TAG}"
        }
        failure {
            echo 'Deployment failed. Check the Jenkins console and Kubernetes events before retrying.'
        }
    }
}
