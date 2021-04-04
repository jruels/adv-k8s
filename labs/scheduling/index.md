# Advanced Scheduling
## Node Affinity/Anti-Affinity 

Get list of nodes
```
kubectl get nodes 
```
You should have 3 nodes
```
NAME              STATUS    ROLES     AGE       VERSION
ip-10-0-100-102   Ready     <none>    2d        v1.9.0
ip-10-0-100-134   Ready     master    2d        v1.9.0
ip-10-0-100-70    Ready     <none>    2d        v1.9.0
```


Set a label on a node that does NOT have 'master' role.
```
kubectl label nodes <ip-10-0-100-70> disktype=ssd
```

Check to make sure node label was applied 
```
kubectl get nodes --show-labels
```

```
NAME              STATUS    ROLES     AGE       VERSION   LABELS
ip-10-0-100-102   Ready     <none>    2d        v1.9.0    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=ip-10-0-100-102
ip-10-0-100-134   Ready     master    2d        v1.9.0    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=ip-10-0-100-134,node-role.kubernetes.io/master=
ip-10-0-100-70    Ready     <none>    2d        v1.9.0    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=ip-10-0-100-70
```

Update YAML with `nodeSelector`
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

Deploy application:
```
kubectl create -f manifests/ssd_pod.yaml
```

Confirm POD was assigned to node with label 

```
kubectl get pods -o wide
```

Now we are going to use `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution` along with `weight` to customize where our Pods are deployed. 

First let's deploy our demo app using `node-affinity.yaml` and then check to see which nodes it was scheduled to. 
```
kubectl get pods
```

```
NAME                             READY   STATUS    RESTARTS   AGE
node-affinity-589d989958-8xtf9   0/1     Pending   0          13s
node-affinity-589d989958-b8hfg   0/1     Pending   0          13s
node-affinity-589d989958-d6hrq   0/1     Pending   0          13s
```

Well, they aren't being scheduled to any nodes.  Do some investigation and determine why. 

Once you have figured out why the Pods are not being scheduled resolve the issue and redeploy to 2 of your nodes.   

After confirming the Pods have been deployed successfully, apply the `preferred` label, to one of the nodes, and then delete the Pod replica running on the other node and you will see it get scheduled on to the node that now has 2 labels applied. 

Now if we check we can see that the Pod was deployed to the node with both labels applied. 
```
 kubectl get pods -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP            NODE                                          NOMINATED NODE
node-affinity-589d989958-9q668   1/1     Running   0          1m    100.96.1.15   ip-172-20-55-161.us-west-1.compute.internal   <none>
node-affinity-589d989958-q7qhh   1/1     Running   0          1m    100.96.1.14   ip-172-20-55-161.us-west-1.compute.internal   <none>
node-affinity-589d989958-tnvlq   1/1     Running   0          1m    100.96.1.13   ip-172-20-55-161.us-west-1.compute.internal   <none>
``` 

## Cleanup
Delete nginx POD
```
kubectl delete pod nginx 
```

## Pod Affinity/Anti-Affinity
### More Useful Example 
Kubernetes has built-in node labels that can be used without being applied manually first.

Interpod Affinity and AntiAffinity can be even more useful when they are used with higher level collections such as ReplicaSets, Statefulsets, Deployments, etc. One can easily configure that a set of workloads should be co-located in the same defined topology, eg., the same node.

### Always co-located on the same node
#### Allow workloads to run on Master node. 
Prior to this lab we must remove the taint from the master so that workloads can run on it. 
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

In a three node cluster, a web application has in-memory cache such as redis. We want the web-servers to be co-located with the cache as much as possible. Here is the yaml snippet of a simple redis deployment with three replicas and selector label `app=store`. The deployment has `PodAntiAffinity` configured to ensure the scheduler does not co-locate replicas on a single node.

Deploy redis YAML
```
kubectl apply -f manifests/affinity.yaml
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
       app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```

The below yaml snippet of the webserver deployment has podAntiAffinity and podAffinity configured. This informs the scheduler that all its replicas are to be co-located with pods that have selector label `app=store`. This will also ensure that each web-server replica does not co-locate on a single node.

Now let’s deploy the YAML for web servers.

```
kubectl apply -f manifests/anti_and_affinity.yaml
```

```
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.12-alpine
```

After creating the above two deployments, our three node cluster should look like below.


|	node-1			|		node-2			|		node-3	|		
|	webserver-1		|	    webserver-2		|	  webserver-3 |
|	    cache-1		|		cache-2			|	      cache-3      |

As you can see, all 3 replicas of the web-server are automatically co-located with the cache as expected.

To confirm run: 
`kubectl get pods -o wide`

Best practice is to configure these highly available stateful workloads such as Redis with AntiAffinity rules for more guaranteed spreading.

### Simplified example
To help understand what just happened here is a simplified example of using `podAffinity` and `podAntiAffinity`. 

Start by looking in `02-affinity/manifests` and you'll see `pod-affinity.yaml`. This manifest creates 2 deployments, a simple demo application and a `redis` datastore. 

The manifest deploys `pod-affinity-1` without any special labels, but if you look at `pod-affinity-2` you'll see that it is required to run on the same node as `pod-affinity-1`.   

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-affinity-1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: pod-affinity-1
    spec:
      containers:
      - name: k8s-demo
        image: aslaen/k8s-demo
        ports:
        - name: nodejs-port
          containerPort: 3000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-affinity-2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: pod-affinity-2
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                    - pod-affinity-1
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis
        image: redis
        ports:
        - name: redis-port
          containerPort: 6379
```

Go ahead and create it
```
kubectl apply -f manifests/pod-affinity.yaml
```

Now if you go check the pods you will see they are running on the same node.  

Scale `pod-affinity-2` and confirm it ends up with `pod-affinity-1`
```
kubectl scale --replicas=4 deployment/pod-affinity-2
```

Delete the deployment 
```
kubectl delete -f manifests/pod-affinity.yaml
```

### Never co-located on the same node

In the same directory there is a `pod-anti-affinity.yaml` file 

Again we are deploying `pod-affinity-1` without any special labels, but this is not the case for our other 2 deployments. 

* Deployment `pod-affinity-2` is set to run on the same node as `pod-affinity-1`
* Deployment `pod-affinity-3` is set to run on a different node than `pod-affinity-1`.

Now deploy and confirm it works. 
```
kubectl apply -f manifests/pod-anti-affinity.yaml
```

You will now see that 4 deployments were created.  

For our 2 deployments we can see they were scheduled to the same node. 

For our 3rd deployment we can see that it was scheduled to a node that did not have `pod-affinity-1` running on it. 

Confirm that the Pods are scheduled as defined. 
```
kubectl get pods -o wide 
```

```
NAME                              READY   STATUS        RESTARTS   AGE   IP            NODE                                          NOMINATED NODE
pod-affinity-1-5dbcfbc86d-4gmz4   1/1     Running       0          15s   100.96.1.16   ip-172-20-55-161.us-west-1.compute.internal   <none>
pod-affinity-1-5dbcfbc86d-97shv   1/1     Terminating   0          46s   100.96.3.17   ip-172-20-41-46.us-west-1.compute.internal    <none>
pod-affinity-2-7c6f9d5f6d-4fblg   1/1     Running       0          15s   100.96.1.17   ip-172-20-55-161.us-west-1.compute.internal   <none>
pod-affinity-2-7c6f9d5f6d-dkxn5   1/1     Running       0          15s   100.96.3.18   ip-172-20-41-46.us-west-1.compute.internal    <none>
pod-affinity-2-7c6f9d5f6d-kvrw2   1/1     Running       0          15s   100.96.1.18   ip-172-20-55-161.us-west-1.compute.internal   <none>
pod-affinity-3-5599479b6d-7w6qs   1/1     Running       0          15s   100.96.2.15   ip-172-20-59-55.us-west-1.compute.internal    <none>
pod-affinity-3-5599479b6d-nfxx9   1/1     Running       0          15s   100.96.2.16   ip-172-20-59-55.us-west-1.compute.internal    <none>
pod-affinity-3-5599479b6d-pdpgg   1/1     Running       0          15s   100.96.2.14   ip-172-20-59-55.us-west-1.compute.internal    <none>
pod-affinity-4-6496648487-9mbtd   0/1     Pending       0          15s   <none>        <none>                                        <none>
```

You'll notice that `pod-affinity-4` may be in a `Pending` status.  

Depending on the lab environment configuration you may not have enough nodes for it to run. To resolve this issue we need to change `requiredDuring...` to `preferredDuring...` and we've already prepared that in `pod-anti-affinity-5.yaml`

Deploy using `pod-anti-affinity-5.yaml` an confirm it schedules property. 

## Visual example 

Highly Available database statefulset has one master and three replicas, one may prefer none of the database instances to be co-located on the same node.

Using Anti-Affinity we can specify that each of the DB-REPLICAs will be on separate nodes.

|	node-1		|	node-2		|	node-3		|	node-4  |
|    DB-MASTER	 |   DB-REPLICA-1	 |   DB-REPLICA-2	 |    DB-REPLICA-3   |


## Cleanup
Delete deployments
```
kubectl delete deployment redis-cache web-server
kubectl delete deployment pod-affinity-1 pod-affinity-2 pod-affinity-3 pod-affinity-4 pod-affinity-5
```

## Congrats!

