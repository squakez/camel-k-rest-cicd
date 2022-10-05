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

Install the pipeline:

```
kubectl apply -f ci/pipeline.yaml -n development
```

Run the pipeline:

```
kubectl apply -f ci/pipeline-run.yaml -n development
```

### Monitor the pipeline

```
kubectl get pipelinerun pipeline-cicd-run -w -n development
```

Check pipeline details (ie, when errors):

```
kubectl get pipelinerun pipeline-cicd-run -o yaml -n development
```
