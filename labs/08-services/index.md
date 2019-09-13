# Kubernetes Advanced Course - Lab 07
# Exposing deployments using Services
By default when you create a deployment in Kubernetes it is only accessible locally.  To make the service available externally we have to expose it and map the container port to the NodePort.

## Create nginx Pod 
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:
  containers:
  - image: nginx
    ports:
      - containerPort: 80
    imagePullPolicy: IfNotPresent
    name: nginx
```

```
kubectl create -f manifests/nginx.yaml
```

## Create a service 
```
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
    - port: 80
  type: NodePort
  selector:
app: nginx
```

```
kubectl create -f manifests/nginx_service.yaml
```

## Confirm external connectivity
Now that we’ve created a POD and service let’s confirm we can connect to it from outside of the container network. 

Get the POD information 
```
kubectl get pods -o wide nginx 
```

```
NAME      READY     STATUS    RESTARTS   AGE       IP          NODE
nginx     1/1       Running   0          6m        10.36.0.1   ip-10-0-100-102
```
Note which NODE the nginx service is running on.

Get service information 
```
kubectl get svc nginx 
``` 

```
NAME      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx     NodePort   10.98.228.135   <none>        80:31897/TCP   5m
```
Note the `NodePort` for the next step

Connect using `curl` to the PUBLIC IP of any node in the cluster on the `NodePort` from previous step. 
```
curl http://<NODE_IP>:<NODE_PORT>
```

## Expose deployment

## Create nginx deployment
```
kubectl create -f manifests/nginx_deployment.yaml
```

Confirm PODS are running as expected 
```
kubectl get pods -o wide -l app=nginx
```

```
NAME                                READY     STATUS    RESTARTS   AGE       IP          NODE
nginx                               1/1       Running   0          20m       10.36.0.1   ip-10-0-100-102
nginx-deployment-6c54bd5869-f4j98   1/1       Running   0          10m       10.36.0.2   ip-10-0-100-102
nginx-deployment-6c54bd5869-lpvrq   1/1       Running   0          10m       10.44.0.3   ip-10-0-100-70
nginx-deployment-6c54bd5869-twfhf   1/1       Running   0          10m       10.44.0.1   ip-10-0-100-70
```

## Create service to expose nginx 
```
kubectl create -f manifests/nginx_deployment_svc.yaml
```

## Get service info
```
kubectl get svc nginx-deployment
```

## Confirm external access to service 
Now curl each of the nodes PUBLIC_IPs the deployment is running on
