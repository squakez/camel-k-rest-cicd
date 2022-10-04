# Camel K CICD Integration

This repository contains a CICD pipeline to run a Camel K integration using Tekton.

# Tekton installation

You can install Tekton pipelines operator following the [Tekton official installation guide](https://tekton.dev/docs/pipelines/install/#installing-tekton-pipelines-on-kubernetes).

## Camel K operator installation

```
kubectl create namespace development
kubectl create namespace production
kamel install -n development
kamel install -n production
```

NOTE: make sure development and production operators share the same container registry.

## Database preparation

Development environment:

```
kubectl apply -f db/conf-dev.yaml -n development
kubectl create secret generic my-datasource --from-file db/datasource-dev.properties -n development
```

Production environment:

```
kubectl apply -f db/conf-prod.yaml -n production
kubectl create secret generic my-datasource --from-file db/datasource-prod.properties -n production
```

NOTE: the database settings are not meant to be used in any production environment without applying proper security policies.

### Initialize DB

Development:

```
kubectl exec -it postgres-dev-xxxxxxxx-yyyyy -n development -- psql -U postgresadmin --password postgresdb -c 'CREATE TABLE customers (name TEXT PRIMARY KEY, city TEXT)'
```

Production:

```
kubectl exec -it postgres-prod-xxxxxxxx-yyyyy -n production -- psql -U postgresadmin --password postgresdb -c 'CREATE TABLE customers (name TEXT PRIMARY KEY, city TEXT)'
```

### Clean test db utility

```
kubectl exec -it postgres-dev-ddddc75cb-gxzff -n development -- psql -U postgresadmin --password postgresdb -c 'delete from customers'
```

## Camel K Service account privileges

We will create a `service account` in the development namespace in order to be able to work both on `development` and `production`:

```
kubectl apply -f ci/sa.yaml -n development
kubectl apply -f ci/rolebinding.yaml -n development
```

## Run the pipeline

Install the [`git-clone` task from Tekton hub](https://hub.tekton.dev/tekton/task/git-clone) int `development` namespace: 

```
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.8/git-clone.yaml -n development
```

Run the pipeline:

```
kubectl apply -f ci/pipeline.yaml -n development
```

### Monitor the pipeline

```
kubectl get pipelinerun pipeline-cicd-run -w -n development
```

Check pipeline details (ie, when errors):

```
kubectl get pipelinerun pipeline-cicd-run -o yaml -n development
```

## Continuos Deployment

Solution based on the blog posted at https://www.arthurkoziel.com/tutorial-tekton-triggers-with-github-integration/

### Install triggers

https://tekton.dev/docs/triggers/install/

kubectl apply -f cd-triggers-sa.yaml
kubectl apply -f gh-interceptor-secret.yaml

NOTE: you may want to enable the addon on minikube via `minikube addons enable ingress`

kubectl apply -f wh-ingress.yaml

kubectl get ingress

### Test webhook locally

cat webhook-sample.json | http post http://192.168.49.2:32698/my-cd-pipeline \
    X-GitHub-Delivery:68b4bf58-4325-11ed-8953-e63aa016e438 \
    X-GitHub-Event:push \
    X-GitHub-Hook-ID:382649378 \
    X-GitHub-Hook-Installation-Target-ID:544844898 \
    X-GitHub-Hook-Installation-Target-Type:repository \
    X-Hub-Signature:sha1=085ba71fa32900cb3d86c75ec9777c77cb0614fb \
    X-Hub-Signature-256:sha256=e306e4533c150dac86b0bb742bf11d7205d0cc9c6806faeb50df4b4c13de4cfc
