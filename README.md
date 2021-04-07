# Banking as a Service POC

## Docker Desktop setup

Bring up the local development environment.

### Docker Registry

When we build Docker images we need a repository to push to - for development purposes it is best to deploy a local repo for speed and prevent pollution of your company's resources.

~~~bash
#
# Install a local Docker registry - access via http://localhost:8080
$ docker-compose up -d
~~~

Next, *add* the following configuration to your Docker Daemon and restart Docker:

~~~json
{
    "insecure-registries" : ["host.docker.internal:5000"]
}
~~~

Verify the registry:

~~~bash
$ docker info
 Insecure Registries:
  host.docker.internal:5000
  127.0.0.0/8
~~~

### Continuous Integration

~~~bash
#
# Install Concourse
$ helm repo add concourse https://concourse-charts.storage.googleapis.com/ && helm repo update
$ helm install concourse concourse/concourse -f config/concourse-helm-values.yml

# Expose the port for web UI and API -- reach via http://localhost:8081 (admin/secret)
$ kubectl port-forward svc/concourse-web -n default 8081:8081

# Install CLI and login
$ brew install --cask fly
$ fly --target=docker-desktop login --concourse-url=http://127.0.0.1:8081 --username=admin --password=secret
~~~

### Continuous Deployment

~~~bash
#
# Install ArgoCD
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Expose the port for web UI and API -- reach via http://localhost:8082 (admin/secret)
$ kubectl port-forward svc/argocd-server -n argocd 8082:443

# Install CLI and login
$ brew install argocd
$ argocd login localhost:8082 \
    --insecure \
    --username admin \
    --password $(kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2)

# Set the admin password
$ argocd account update-password \
    --insecure \
    --current-password $(kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2) \
    --new-password secret
~~~

### Test

~~~bash
# Set some constants
$ ARGOCD_APP=simple-app
$ ARGOCD_TOKEN=$(curl -s --insecure https://localhost:8082/api/v1/session -d $'{"username":"admin","password":"secret"}' | jq -r .token)

# Deploy build pipeline - pushes to Docker registry and syncs app in ArgoCD
$ fly -t docker-desktop set-pipeline -p simple-pipeline \
    -c apps/simple-app/pipeline.yml \
    --var image-repo-name=host.docker.internal:5000 \
    --var argocd_token=$ARGOCD_TOKEN \
    --var argocd_app=$ARGOCD_APP

# Deploy the deployment pipeline 
$ argocd app create $ARGOCD_APP \
    --insecure \
    --repo https://github.com/robinhuiser/baas-poc.git \
    --path apps/simple-app/overlays/docker-desktop \
    --dest-namespace default \
    --dest-server https://kubernetes.default.svc

# Unpause and kickoff the build in Concourse
$ fly -t docker-desktop unpause-pipeline -p simple-pipeline
$ fly -t docker-desktop trigger-job -j simple-pipeline/build-push-deploy
~~~

## Tear all down

~~~bash
# Stop the proxies first!
$ kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
$ kubectl delete namespaces argocd
$ helm uninstall concourse
$ kubectl delete namespaces concourse-main
$ docker-compose down --remove-orphans --volumes
~~~
