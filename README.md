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
kubectl apply -f db/conf-dev.yaml
kubectl create secret generic my-datasource --from-file datasource-dev.properties -n development
```

Production environment:

```
kubectl apply -f db/conf-prod.yaml
kubectl create secret generic my-datasource --from-file datasource-prod.properties -n production
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

```
kubectl apply -f ci/sa.yaml -n development
kubectl apply -f ci/rolebinding.yaml -n development
```

## Run the pipeline

```
kubectl apply -f ci/pipeline.yaml -n development
```

### Monitor the pipeline

```
kubectl get pipelinerun pipeline-cicd-run -w -n development
```