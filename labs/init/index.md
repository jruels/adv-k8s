# Kubernetes Advanced Course Lab 10
# Init containers
In this lab we will be exploring init containers and how they can be used for complex application deployment orchestration. 

## Create init container POD
First we need to create a POD container our init container and application. 
Create `init.yaml` on your Kubernetes master with the following: 
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```

Now let’s go ahead and create this POD. 
```
kubectl apply -f init.yaml
```

Check to see what status this POD is in. 
```
kubectl get -f init.yaml
```

You should see something like the following 
```
NAME        READY     STATUS     RESTARTS   AGE
myapp-pod   0/1       Init:0/2   0          8s
```

To dive a little deeper check out the events 
```
kubectl describe -f init.yaml
```

Pay attention to the `State` of the `init-mydb` container 

Let’s check the logs and see if they provide any insight into the issue. 
```
kubectl logs myapp-pod -c init-myservice
``` 

The log output should show something similar to: 
```
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

waiting for myservice
nslookup: can't resolve 'myservice'
```

## Create required services
Ok, now we know that it’s not starting because the init POD is waiting on a couple services. Let’s go ahead and create file `init_services.yaml` containing the following
```
kind: Service
apiVersion: v1
metadata:
  name: myservice
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
---
kind: Service
apiVersion: v1
metadata:
  name: mydb
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9377
```

Now create it 
```
kubectl apply -f init_services.yaml
```

Check on your init POD 
```
kubectl get -f init.yaml
```

Everything should have started up as expected 
```
NAME        READY     STATUS    RESTARTS   AGE
myapp-pod   1/1       Running   0          2m
```

So, in summary an init container is used to confirm something has completed prior to the POD being started. In this case we checked to make sure a couple services were available before we started up our application POD. 

## Nginx init container 
This exercise will show how an init container can be used to fetch a file and provide it to a web server using persistent volumes.

## Create nginx POD
Let’s create a file called `init-nginx.yaml` containing the following
```
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html
  # These containers are run during pod initialization
  initContainers:
  - name: install
    image: busybox
    command:
    - wget
    - "-O"
    - "/work-dir/index.html"
    - http://kubernetes.io
    volumeMounts:
    - name: workdir
      mountPath: "/work-dir"
  dnsPolicy: Default
  volumes:
  - name: workdir
    emptyDir: {}
```

In the configuration file, you can see that the Pod has a Volume that the init container and the application container share.

The init container mounts the shared Volume at `/work-dir`, and the application container mounts the shared Volume at `/usr/share/nginx/html`. The init container runs the following command and then terminates:
```
wget -O /work-dir/index.html http://kubernetes.io
```
Notice that the init container writes the `index.html` file in the root directory of the nginx server.

Create the POD
```
kubectl apply -f init-nginx.yaml
```

Verify that the nginx container is running:
```
kubectl get pod init-demo
```

The output shows that the nginx container is running:
```
NAME        READY     STATUS    RESTARTS   AGE
init-demo   1/1       Running   0          1m
```

Now let’s shell into the nginx container running in the init-demo Pod:
```
kubectl exec -it init-demo -- /bin/bash
```

Send a GET request to the nginx web server 
```
apt-get update && apt-get install -y curl 
curl -s localhost | grep "is open source"
```

The output shows that nginx is serving the web page that was written by the init container:
```
<p>Kubernetes is open source giving you the freedom to take advantage of on-premises, hybrid, or public cloud infrastructure, letting you effortlessly move workloads to where it matters to you.</p>
```

We can also confirm the `install` init container completed its task by looking at the logs. 
```
kubectl logs -f init-demo -c install
```

Look for something similar to
```
Connecting to kubernetes.io (45.54.44.100:80)
Connecting to kubernetes.io (45.54.44.100:443)
index.html           100% |*******************************| 21693   0:00:00 ETA
```

## Lab Complete 
