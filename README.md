# Camel K CICD Integration

This repository contains a CICD pipeline to run a Camel K integration using Tekton.

# Tekton installation

You can install Tekton pipelines operator following the [Tekton official installation guide](https://tekton.dev/docs/pipelines/install/#installing-tekton-pipelines-on-kubernetes).

# Camel K operator installation

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

NOTE: the database settings are not meant to be used in any production environment without applying proper security policies. The Pod running is ephemeral, so, anything stored there will be lost when restarting.

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

# Continuos Deployment

Solution based on the blog posted at https://www.arthurkoziel.com/tutorial-tekton-triggers-with-github-integration/

### Install triggers

https://tekton.dev/docs/triggers/install/

kubectl apply -f cd-triggers-sa.yaml -n development

NOTE: you may want to enable the addon on minikube via `minikube addons enable ingress`

kubectl apply -f wh-ingress.yaml -n development

kubectl get ingress / minikube service list (to simuate a webhook post locally)

kubectl apply -f cd-triggers.yaml -n development

Watch out for any new pipeline triggered:

k get pipelinerun -n development -w

Make change and push to the github repo.

```
{"severity":"info","timestamp":"2022-10-04T11:02:30.537Z","logger":"eventlistener","caller":"sink/sink.go:409","message":"ResolvedParams : []","eventlistener":"github-push","namespace":"development","/triggers-eventid":"b7fda1d3-044e-402a-b2c2-900afd849681","eventlistenerUID":"98b1322c-1793-425a-8577-8fec3e708b3f","/triggers-eventid":"b7fda1d3-044e-402a-b2c2-900afd849681","/trigger":"github-listener"}
{"severity":"info","timestamp":"2022-10-04T11:02:30.539Z","logger":"eventlistener","caller":"resources/create.go:98","message":"Generating resource: kind: &APIResource{Name:pipelineruns,Namespaced:true,Kind:PipelineRun,Verbs:[delete deletecollection get list patch create update watch],ShortNames:[pr prs],SingularName:pipelinerun,Categories:[tekton tekton-pipelines],Group:tekton.dev,Version:v1beta1,StorageVersionHash:RcAKAgPYYoo=,}, name: github-push-pipeline-run-"}
{"severity":"info","timestamp":"2022-10-04T11:02:30.539Z","logger":"eventlistener","caller":"resources/create.go:106","message":"For event ID \"b7fda1d3-044e-402a-b2c2-900afd849681\" creating resource tekton.dev/v1beta1, Resource=pipelineruns"}
```

### Test webhook locally

http post http://192.168.49.2:32698/my-cd-pipeline X-GitHub-Event:push json=test 
    
