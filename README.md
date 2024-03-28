# Setting up a development environment on multi-node Kind Kubernetes cluster

This repo is designed for use with my [Medium article](https://medium.com/@martin.hodges/using-kind-to-develop-and-test-your-kubernetes-deployments-54093692c9fa) 
on using Kind to develop and test your Kubernetes deployments.

Applying the steps described in the article, you will end up with:

- 1 master node
- 2 worker nodes
- a sample application
- Istio service mesh
- Grafana/Loki/Prometheus stack
- Postgres database

You will need to install Kind, Docker and Helm.

Create your multi-node cluster:
    kind create cluster --config kind-config.yml

Create a sample Docker image and preload it:
    cd sample-app 
    docker build -t sample-app:1.0 .
    kind load docker-image sample-app:1.0

Deploy the sample app:
    kubectl apply -f my-deployment.yml
    cd ..

You can now access this sample application at: [http://localhost:30000](http://localhost:30000)

To install the applications, you need a number of Helm charts in your repository:
    helm repo add istio https://istio-release.storage.googleapis.com/charts
    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo add cnpg https://istio-release.storage.googleapis.com/charts
    helm repo update

Install Istio service mesh:
    kubectl create namespace istio-system
    helm install istio-base istio/base -n istio-system --set defaultRevision=default
    helm install istiod istio/istiod -n istio-system --wait
    kubectl label namespace default istio-injection=enabled --overwrite

Install the Grafana, Loki, Prometheus stack:
    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: Namespace
    metadata:
      name: monitoring
      labels:
        istio-injection: enabled
    EOF
    helm install loki grafana/loki-stack -n monitoring -f loki-config.yml
    kubectl apply -f grafana-svc.yml

Get the login credentials:
    kubectl get secret loki-grafana -n monitoring -o jsonpath={.data.admin-user} | base64 -d; echo
    kubectl get secret loki-grafana -n monitoring -o jsonpath={.data.admin-password} | base64 -d; echo

You can now access Grafana at [http://localhost:31300](http://localhost:31300) and log in with the
credentials you just found.

Install the Postgres database operator:
    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: Namespace
    metadata:
      name: pg
      labels:
        istio-injection: enabled
    EOF
    helm install cnpg cnpg/cloudnative-pg -n pg
    kubectl apply -f db-user-config.yml
    kubectl apply -f db-config.yml

You should now be able to access the database at localhost on port 31321 with the credentials in
the db-user-config file (remember to decode the base 64 strings).

