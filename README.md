## Wordpress + MySQL + EFK(ElasticSearch + Fluentd + Kibana) + Portworx

This repository contains example code for Deploy - Wordpress + MySQL + EFK(ElasticSearch + Fluentd + Kibana) + Portworx:

- Portworx
- Wordpress (Using Storage Portworx)
- MySQL (Using Storage Portworx)
- ElasticSearch (Using Elastic Cloud on Kubernetes - ECK)
- Fluentd + Fluentbit (Using Helm chart Banzaicloud logging-operator-logging)
- Logging Flow (Using Helm Chart Banzaicloud logging-operator)

## Deploy instructions

### 1. Create Namespaces

```
kubectl create ns site-wordpress
kubectl create ns cert-manager
kubectl create ns ingress-nginx
kubectl create ns elastic-system
kubectl create ns observability
```

### 2. Deploy Helm Charts

```
Banzaicloud logging-operator and logging-operator-logging

helm repo add banzaicloud-stable https://kubernetes-charts.banzaicloud.com
helm repo update
helm install logging-operator banzaicloud-stable/logging-operator -n observability \
  --set createCustomResource=false \
  --set rbac.enable=true


Cert-Manager for TLS Certificates

helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager -n cert-manager \
  --namespace cert-manager \
  --version v0.16.1 \
  --set installCRDs=true


Ingress Nginx

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx
```

### 3. Deploy Portworx

```
kubectl apply -f manifests/portworx/portworx_enterprise.yaml
```

### 4. Deploy ECK Operator (For ElasticSearch & Kibana)

```
kubectl apply -f https://download.elastic.co/downloads/eck/1.2.1/all-in-one.yaml
```

### 5. Deploy MySQL

For testing purposes a password will be stored in Kubernetes Secrets, but for production grade deploy, consider to use some secret-manager solution.

```
kubectl create secret generic mysql-pass --from-file=./password.txt -n site-wordpress
kubectl apply -f manifests/mysql/mysql-sc.yaml -n site-wordpress
kubectl apply -f manifests/mysql/mysql-pvc.yaml -n site-wordpress
kubectl apply -f manifests/mysql/mysql-svc.yaml -n site-wordpress
kubectl apply -f manifests/mysql/mysql-deployment.yaml -n site-wordpress
```

### 6. Setup Cert-Manager and create Certificate

For testing purposes Certificate and DNS provider secrets will be stored in this Kubernetes Secret, but for production grade deploy, consider to use some secret-manager solution.

```
kubectl apply -f manifests/cert-manager/issuer.yaml -n site-wordpress
kubectl apply -f manifests/cert-manager/certificate.yaml -n site-wordpress
```

### 7. Deploy Wordpress

```
kubectl apply -f manifests/wordpress/wordpress-sc.yaml -n site-wordpress
kubectl apply -f manifests/wordpress/wordpress-pvc.yaml -n site-wordpress
kubectl apply -f manifests/wordpress/wordpress-svc.yaml -n site-wordpress
kubectl apply -f manifests/wordpress/wordpress-deployment.yaml -n site-wordpress
kubectl apply -f manifests/wordpress/wordpress-ingress.yaml -n site-wordpress
```

### 8. Setup ElasticSearch Cluster and Kibana using ECK CRDs

Actually with ECK isn't possible to set the password by CRDs. But, thre is a [workarround](https://github.com/elastic/cloud-on-k8s/issues/967#issuecomment-497636249) creating the {clusterName}-es-elastic-user Secret before creating the Elasticsearch resource. For testing purposes a password will be configured in this way, but for production grade deploy, consider to use some secret-manager solution.

```
kubectl create secret generic logging-es-elastic-user --from-literal=elastic=teste -n observability
kubectl apply -f manifests/efk/es-sc.yaml -n observability
kubectl apply -f manifests/efk/es-cluster.yaml -n observability
kubectl apply -f manifests/efk/kibana-eck.yaml -n observability
```

### 9. Setup Logging Flow using Banzaicloud logging-operator CRDs
```
kubectl apply -f manifests/efk/logging-operator-logging.yaml -n observability
kubectl apply -f manifests/efk/logging-operator-cluster-output.yaml -n observability
kubectl apply -f manifests/efk/logging-operator-flow.yaml -n site-wordpress
```

PS: For testing purposes a script for create cluster in GKE has been added in `manifests/cluster-example`.