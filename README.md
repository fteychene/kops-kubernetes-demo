# Purpose

This demo have been made to :
* Start a [Kubernetes](https://kubernetes.io/) cluster in AWS using [kops](https://github.com/kubernetes/kops)
* Install some tools on the cluster ([Dashboard](https://github.com/kubernetes/dashboard), [Heapster](https://github.com/kubernetes/heapster), [WeaveScope](https://www.weave.works/oss/scope/))
* Install some components using [Helm](https://helm.sh/)
* Install [SockShop](https://microservices-demo.github.io/) microservice demo application

# Requirements
To run this demo you need to install the following binaries in your environment :
* [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [`kops`](https://github.com/kubernetes/kops/blob/master/docs/install.md)
* [`awscli`](http://docs.aws.amazon.com/fr_fr/cli/latest/userguide/installing.html)
* [`helm`](https://github.com/kubernetes/helm/blob/master/docs/install.md)

# Setup

This demo use a subdomain managed in AWS Route53 for the cluster.
To create this subdomain I use this configuration as described in [kops configuration guide](https://github.com/kubernetes/kops/blob/master/docs/aws.md#scenario-1b-a-subdomain-under-a-domain-purchasedhosted-via-aws)

```bash
export PARENT_DOMAIN="fteychene.xyz"
export KUBERNETES_DOMAIN="kubernetes.$PARENT_DOMAIN"
export KOPS_BUCKET_NAME="kubernetes-fteychene-state-store"

```
## Subdomain 
Store the parent domain AWS id
```bash
export PARENT_ID=$(aws route53 list-hosted-zones | jq '.HostedZones[] | select(.Name==env.PARENT_DOMAIN+".") | .Id' | cut -d/ -f3 | cut -d\" -f1)
```

Create sub domain and get NameServers
```bash
ID=$(uuidgen) && aws route53 create-hosted-zone --name $KUBERNETES_DOMAIN --caller-reference $ID | jq .DelegationSet.NameServers
```

Change the nameservers in the [subdomain.json]() file by the one returned by the command.

Apply subdomain routing in parent hosted zone
```bash
aws route53 change-resource-record-sets --hosted-zone-id $PARENT_ID --change-batch file://subdomain.json
```

## Create S3 bucket for kops configuration storing
```bash
aws s3api create-bucket --bucket $KOPS_BUCKET_NAME --create-bucket-configuration LocationConstraint=eu-west-1 --region eu-west-1
```

Add versionning for state to rollback modifications if needed
```bash
aws s3api put-bucket-versioning --bucket $KOPS_BUCKET_NAME  --versioning-configuration Status=Enabled
```

# Cluster creation

## Create config
```bash
export NAME=kubernetes-test
export KOPS_STATE_STORE="s3://$KOPS_BUCKET_NAME"

kops create cluster \
    --cloud=aws \
    --node-count 3 \
    --zones eu-west-1a,eu-west-1b,eu-west-1c \
    --master-zones eu-west-1a,eu-west-1b \
    --master-count=3 \
    --dns-zone $KUBERNETES_DOMAIN \
    --node-size t2.medium \
    --master-size t2.medium \
    --networking weave \
    $NAME
```

## Edit configuration
```bash
kops edit cluster ${NAME}
```

## Create cluster
```bash
kops update cluster ${NAME} --yes
```

# Configuration cluster

## Deploy heapster
```bash
kubectl create -f heapster/
```

## Deploy dashboard
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
# Add admin right to dashboard service account (should not be done in real env)
kubectl apply -f dashboard-admin-role.yaml
kubectl proxy &
```
You can access to the deployed dashboard at http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

## Deploy weave scope
```bash
kubectl apply --namespace kube-system -f "https://cloud.weave.works/k8s/scope.yaml?k8s-version=$(kubectl version | base64 | tr -d '\n')"
kubectl port-forward -n kube-system "$(kubectl get -n kube-system pod --selector=weave-scope-component=app -o jsonpath='{.items..metadata.name}')" 4040 &
```
Weave scope is available at http://localhost:4040

## Install tiller (helm)
```bash
helm init
```

# Apps

## Helm Jenkins
```bash
helm install --name jenkins stable/jenkins
printf $(kubectl get secret --namespace default jenkins-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```
You can find the external address by running the command `kubectl get svc jenkins-jenkins`

## Helm Graphana
```bash
helm install --name grafana stable/grafana
kubectl get secret --namespace default grafana-grafana -o jsonpath="{.data.grafana-admin-password}" | base64 --decode ; echo
kubectl --namespace default port-forward $(kubectl get pods --namespace default -l "app=grafana-grafana,component=grafana" -o jsonpath="{.items[0].metadata.name}") 3000 &
```
Grafana will be aviable on http://localhost:3000

## Sock shop sample
```bash
kubectl create namespace sock-shop
kubectl apply -f prezkube.tabmo.net/sock-shop.yaml --namespace sock-shop
```

You can find the external address by running the command `kubectl -n sock-shop get svc sock-shop`