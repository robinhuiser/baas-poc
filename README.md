# Banking as a Service POC

Brining up the local development environment.

## Docker Desktop setup

### Docker Registry

When we build Docker images we need a repository to push to - for development purposes it is best to deploy a local repo for speed and prevent pollution of your company's resources.

~~~bash
#
# Install a local Docker registry - access via http://localhost:8080
$ docker-compose up -d
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

# Expose the port for web UI and API -- reach via http://localhost:8080 (admin/secret)
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
$ ARGOCD_TOKEN=$(curl --insecure https://localhost:8082/api/v1/session -d $'{"username":"admin","password":"secret"}' | jq -r .token)

# Deploy build pipeline - pushes to Docker registry and syncs app in ArgoCD
$ fly -t docker-desktop set-pipeline -p simple-pipeline \
    -c apps/simple-app/pipeline.yml \
    --var image-repo-name=host.docker.internal:5000 \
    --var argocd_token=$ARGOCD_TOKEN \
    --var argocd_app=$ARGOCD_APP

# 


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
