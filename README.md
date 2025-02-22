name: Deploy DEV - Pull Request to development

on:
  pull_request:
    branches:
    - development
    paths-ignore:
    - 'README.md'

env:
  PROJECT: medium
  GKE_ZONE: us-west2
  GKE_CLUSTER: meidum
  NAMESPACE: development
  ENV: dev
  GITHUB_SHA: ${{ github.sha }}
  REGISTRY_HOSTNAME: gcr.io
  SERVICE_NAME: mediumpod
  HTTP_PROXY: http://votre-proxy:port
  HTTPS_PROXY: http://votre-proxy:port
  NO_PROXY: localhost,127.0.0.1,.votre-domaine.com

jobs:
  pull_request_workflow:
    name: Deploy DEV - Pull Request to development
    runs-on: self-hosted  # Utilisez votre runner self-hosted
    steps:

    # Clone the repository
    - name: Checkout
      uses: actions/checkout@v2

    # Setup Admin gcloud CLI
    - name: Configure Admin Google Cloud Account
      uses: GoogleCloudPlatform/github-actions/setup-gcloud@master
      with:
        version: '270.0.0'
        service_account_key: ${{ secrets.REGISTRY_SA }}

    # Configure docker to use the gcloud command-line tool as a credential helper
    - name: Configure Docker
      run: |
        mkdir -p ~/.docker
        echo '{
          "proxies": {
            "default": {
              "httpProxy": "$HTTP_PROXY",
              "httpsProxy": "$HTTPS_PROXY",
              "noProxy": "$NO_PROXY"
            }
          }
        }' > ~/.docker/config.json
        gcloud auth configure-docker

    # Build Dockerfile
    - name: Docker build
      run: |
        DOCKER_BUILDKIT=1 docker build \
          --build-arg BUILDKIT_INLINE_CACHE=1 \
          --cache-from $REGISTRY_HOSTNAME/$ADMIN_PROJECT/$SERVICE_NAME:$ENV-latest \
          -t $REGISTRY_HOSTNAME/$ADMIN_PROJECT/$SERVICE_NAME:$ENV-$GITHUB_SHA \
          -t $REGISTRY_HOSTNAME/$ADMIN_PROJECT/$SERVICE_NAME:$ENV-latest \
          -t $REGISTRY_HOSTNAME/$ADMIN_PROJECT/$SERVICE_NAME:$ENV-$GITHUB_RUN_NUMBER \
          .

    # Lint
    - name: Lint
      run: |
        LINT

    # Unit Test
    - name: Unit Test
      run: |
        UNIT

    # Push the Docker image to Google Container Registry
    - name: Publish
      run: |
        docker push $REGISTRY_HOSTNAME/$ADMIN_PROJECT/$SERVICE_NAME:$ENV-$GITHUB_SHA
        docker push $REGISTRY_HOSTNAME/$ADMIN_PROJECT/$SERVICE_NAME:$ENV-latest
        docker push $REGISTRY_HOSTNAME/$ADMIN_PROJECT/$SERVICE_NAME:$ENV-$GITHUB_RUN_NUMBER

    # Setup Env GKE
    - name: Configure Cluster
      run: gcloud container clusters get-credentials $GKE_CLUSTER --zone $GKE_ZONE --project $PROJECT

    # Configuring deploy files
    - name: Configuring Deployment
      run: | 
        sed -i \
          -e 's#newTag: .*#'"newTag: ${ENV}-${GITHUB_SHA}"'#' \
          k8s/overlays/$NAMESPACE/kustomization.yml

    # Deploy to GKE cluster
    - name: Deployment
      run: |
        kubectl apply -k k8s/overlays/$NAMESPACE
        DEPLOYMENT_NAME=`sed -n -e 's/^.*app: //p' k8s/base/kustomization.yml`
        kubectl rollout status deployment/$DEPLOYMENT_NAME -n $NAMESPACE --timeout=6m
