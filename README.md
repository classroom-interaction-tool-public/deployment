# Setup of CIT in a Minikube environment
**Prerequisites:**
* Docker installed
* Minikube installed


First, start-up Minikube by issuing the command 
```
minikube start
```


# 1. RabbitMQ
```
kubectl apply -f rabbitmq-secret.yaml
kubectl apply -f rabbitmq-statefulset.yaml
kubectl apply -f rabbitmq-service.yaml
```
```
kubectl port-forward svc/rabbitmq 15672:15672 5672:5672
```

# 2. Mongo DB setup
The next part is to set-up the sharded Mongo DB environment in Minikube

Start by applying the mongo and mongos service to the cluster

```
kubectl apply -f mongo-service.yaml
kubectl apply -f mongos-service.yaml
```

## 1.1 Shards
Next, let's apply the first shard with its replicas to the cluster 
```
kubectl apply -f mongo-replica-set-1.yaml
```

We can verify that the pods are up and running with the commands ```kubectl get pods``` and ```kubectl get svc```. The output should be something like the following:

**_NOTE:_** It might take some time before the pods are up and running in the cluster

```
❯ kubectl get pods && kubectl get svc
NAME                    READY   STATUS    RESTARTS   AGE
mongo-replica-set-1-0   1/1     Running   0          52s
mongo-replica-set-1-1   1/1     Running   0          33s
mongo-replica-set-1-2   1/1     Running   0          32s
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP     78s
mongo        ClusterIP   None            <none>        27018/TCP   63s
mongos       ClusterIP   10.102.61.131   <none>        27017/TCP   58s
```


Now, let's exec into one of the replica pods to create the connection between all replicas in the shard.
```
kubectl exec -it mongo-replica-set-1-0 -- mongo --host mongo --port 27018
```

The next step is to initiate the connection with other replicas to create the set.

```
rs.initiate({
  _id: 'rs1',
  members: [
    { _id: 0, host: 'mongo-replica-set-1-0.mongo.default.svc.cluster.local:27018' },
    { _id: 1, host: 'mongo-replica-set-1-1.mongo.default.svc.cluster.local:27018' },
    { _id: 2, host: 'mongo-replica-set-1-2.mongo.default.svc.cluster.local:27018' }
  ]
})
```

Now exit out of the pod with the following command

```
exit
```


The next step is to do the same for the second shard with its replica set

```
kubectl apply -f mongo-replica-set-2.yaml
```

Enter one of the pods

```
kubectl exec -it mongo-replica-set-2-0 -- mongo --host mongo --port 27018
```

Initiate the replica set connections

```
rs.initiate({
    _id: 'rs2',
    members: [
      { _id: 0, host: 'mongo-replica-set-2-0.mongo.default.svc.cluster.local:27018' },
      { _id: 1, host: 'mongo-replica-set-2-1.mongo.default.svc.cluster.local:27018' },
      { _id: 2, host: 'mongo-replica-set-2-2.mongo.default.svc.cluster.local:27018' }
    ]
  })
```

exit the pod

```
exit
```

## Config Server
Now, let's set up the config server by first applying it to the cluster

```
kubectl apply -f mongo-config-service.yaml
kubectl apply -f mongo-config-server.yaml
```

Same as with the shards, we need to enter one of the pods to set up the connections

```
kubectl exec -it mongo-config-server-0 -- mongo --port 27019 
```

Initiate the config pod connections
```
  rs.initiate({
    _id: 'rs-config',
    configsvr: true,
    members: [
      { _id: 0, host: 'mongo-config-server-0.mongo-config.default.svc.cluster.local:27019' },
      { _id: 1, host: 'mongo-config-server-1.mongo-config.default.svc.cluster.local:27019' },
      { _id: 2, host: 'mongo-config-server-2.mongo-config.default.svc.cluster.local:27019' }
    ]
  })
```

Exit the pod

```
exit
```

## Router
The final part of the database setup is the router that will handle the requests. Start by applying the mongos router to the cluster

```
kubectl apply -f mongos-deployment.yaml
```

Next, enter the router pod to set up the sharded environment

```
kubectl exec -it $(kubectl get pods -l role=mongos -o jsonpath='{.items[0].metadata.name}') -- mongo 
```

Set up and enable sharding

```
  sh.addShard('rs1/mongo-replica-set-1-0.mongo.default.svc.cluster.local:27018');
  sh.addShard('rs2/mongo-replica-set-2-0.mongo.default.svc.cluster.local:27018');

  use citdb;
  sh.enableSharding('citdb');
```

Finally, exit the pod
```
  exit
```

# API setup

First, we need to set the environment so docker images are build available to the Minikube environment

```
eval $(minikube docker-env) 
```

Next, we can build the image of the API with the following command

```
docker buildx build -t fastify-api:latest ../api
```

Now, let's enable the environment variables and secrets used by the API and make them available to the cluster.

```
kubectl apply -f configmap.yaml
kubectl apply -f fastify-api-secret.yaml
```

Finally, we can apply the API to the Kubernetes cluster.

```
kubectl apply -f api-deployment.yaml
```

To make the server accessible on localhost, issue one of the following commands
```
kubectl port-forward service/fastify-api 5000:5000
```
```
minikube service fastify-api-service
```

# Student API setup
```
eval $(minikube docker-env) 
```

Next, we can build the image of the API with the following command

```
docker buildx build -t api-student:latest ../api-student
```

Now, let's enable the environment variables and secrets used by the API and make them available to the cluster.

```
kubectl apply -f api-configmap.yaml
kubectl apply -f api-secret.yaml
kubectl apply -f api-student-deployment.yaml
```

To make the server accessible on localhost, issue one of the following commands
```
kubectl port-forward service/student-api 5003:5003
```
```
minikube service student-api-service
```

# Client Student setup
```
eval $(minikube docker-env) 
docker buildx build -t client-student:latest ../client-student
kubectl apply -f client-student-deployment.yaml
```

To access the client using localhost, issue one of the following commands
```
kubectl port-forward service/client-student-service 3000:3000
```
```
minikube service client-student-service
```

# Client Teacher setup
```
eval $(minikube docker-env) 
docker buildx build -t client-teacher:latest ../client-teacher
kubectl apply -f client-teacher-deployment.yaml
```

To access the client using localhost, issue one of the following commands
```
kubectl port-forward service/client-teacher-service 3001:3001
```
```
minikube service client-teacher-service
```



# Deleting the environment
If you want a completely fresh start, delete the Minikube environment using the following command
```
minikube delete
```

and follow the guide from the start to set up the environment again.