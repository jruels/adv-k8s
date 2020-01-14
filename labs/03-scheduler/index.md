# Advanced Kubernetes - Lab 03

# Custom Scheduler 

If the Kubernetes scheduler’s various features don’t give you enough control over the scheduling of your workloads, you can delegate responsibility for scheduling arbitrary subsets of pods to your own custom scheduler(s) that run(s) alongside, or instead of, the default Kubernetes scheduler.
Each new pod is normally scheduled by the default scheduler. But if you provide the name of your own custom scheduler, the default scheduler will ignore that Pod and allow your scheduler to schedule the Pod to a node.

## Create a POD YAML with a custom scheduler 
Here we have a POD manifest where we specify the `schedulerName` field:
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  schedulerName: my-scheduler
  containers:
  - name: nginx
    image: nginx:1.10
```

If we create this Pod without deploying a custom scheduler, the default scheduler will ignore it and it will remain in a `Pending` state. So we need a custom scheduler that looks for, and schedules, pods whose `schedulerName` field is `my-scheduler`.

A custom scheduler can be written in any language and can be as simple or complex as you need. Here is a very simple example of a custom scheduler written in Bash that assigns a node randomly. 
Note that you need to run this along with `kubectl proxy` for it to work.

## Deploy nginx POD from manifest above 
```
kubectl apply -f manifests/scheduler.yaml
```

Now confirm it is in a `Pending` state.

```
kubectl get pods 
```

```
NAME      READY     STATUS    RESTARTS   AGE
nginx     0/1       Pending   0          15s
```


## Run custom scheduler

The `scheduler.sh` script runs `kube-proxy` and  creates a tunnel at `http://localhost:9090` which allows connections to the Kubernetes API using `curl`

```
#!/bin/bash
kubectl proxy --port=9090 &
while true;
do
    for PODNAME in $(kubectl get pods -o json | jq '.items[] | select(.spec.schedulerName == "my-scheduler") | select(.spec.nodeName == null) | .metadata.name' | tr -d '"');
    do
        NODES=($(kubectl get nodes -o json | jq '.items[].metadata.name' | tr -d '"'))
        NUMNODES=${#NODES[@]}
        CHOSEN=${NODES[$[ $RANDOM % $NUMNODES ]]}
        curl --header "Content-Type:application/json" --request POST --data '{"apiVersion":"v1", "kind": "Binding", "metadata": {"name": "'$PODNAME'"}, "target": {"apiVersion": "v1", "kind"
: "Node", "name": "'$CHOSEN'"}}' http://localhost:9090/api/v1/namespaces/default/pods/$PODNAME/binding/
        echo "Assigned $PODNAME to $CHOSEN"
    done
    sleep 10
done
```

## Install jq 
The bash script requires `jq` to parse the `JSON` output. 
```
sudo apt-get update && sudo apt-get install -y jq 
```

Now let’s run the scheduler 
```
bash scripts/scheduler.sh &
```

## Confirm POD was scheduled. 
You will now see output from `scheduler.sh` showing the POD was scheduled. 
```
Starting to serve on 127.0.0.1:9090
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Success",
  "code": 201
}Assigned nginx to ip-10-0-100-134
```
Now when we check `kubectl` we can see that the POD has been successfully scheduled and is no longer in `Pending` state. 
```
kubectl get pods 
```

```
NAME      READY     STATUS    RESTARTS   AGE
nginx     1/1       Running   0          5m
```


## Cleanup
Alright now let’s clean everything up.

Kill `kubectl proxy`
```
killall kubectl
```

Delete POD
```
kubectl delete pods nginx
```

# Go Scheduler
The above steps work for a simple test but you don't want to run a shell script every time you are going to schedule an app. To improve on the steps in the previous section we are going to deploy a scheduler to our cluster that watches for unscheduled Pods with `schedulerName: random-scheduler`. 

## Deploy scheduler 

Start by deploying the scheduler
```
kubectl apply -f manifests/random-scheduler.yaml
```

What happened? 

More than likely you will see the Pod never gets created and if you look at running deployments you will see something like 
```
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
random-scheduler   1         0         0            0           1m
```

The above shows that there is 1 Pod that should be running (DESIRED) but it was not scheduled correctly and so 0 are available. 

Do some troubleshooting and see why the Pod was not scheduled. 

Hint: Deployments do not control Pods directly,  look at the replication controller. 

Ok, now that you've figured out the issue delete the deployment 
```
kubectl delete -f manifests/random-scheduler.yaml
```

Look in the manifests directory and you'll see what's required to resolve the above issue.  Use `kubectl` to create it and then deploy the `random-scheduler` again. 

This time you should see it start successfully. 

```
NAME                                READY     STATUS    RESTARTS   AGE
random-scheduler-6b66b47569-d2tsd   1/1       Running   0          98m
```


The logs of the scheduler should show that the nodes in the cluster were added to the informer’s store.
```
kubectl logs -f $(kubectl get pod -l app=random-scheduler -o jsonpath="{.items[0].metadata.name}")
```

Output will be similar to below
```
I'm a scheduler!
2019/01/03 16:03:51 New Node Added to Store: gke-martonseregdgk7-system-d609a14b-lrj0
2019/01/03 16:03:51 New Node Added to Store: gke-martonseregdgk7-pool2-694b3996-3zg3
2019/01/03 16:03:51 New Node Added to Store: gke-martonseregdgk7-pool1-7c15ca6c-s0fg
```

Now that the `random-scheduler` is running, it’s time to try it out. We’ve created an example deployment that contains some sleeping containers, with the `schedulerName` changed in the pod spec.

```
spec:
  schedulerName: random-scheduler
```

Let’s apply this deployment and check if the pods were successfully scheduled. Our pods should be running, and the scheduler logs should show some new entries.

```
kubectl apply -f manifests/sleep.yaml
```

Confirm the Pods were scheduled. 
```
kubectl get pods -l app=sleep
```

You should see something similar to: 
```
NAME                                READY   STATUS    RESTARTS   AGE
sleep-5c78857b77-jdfnp              1/1     Running   0          30s
```

Now check out the scheduler logs
```
kubectl logs -f $(kubectl get pod -l app=random-scheduler -o jsonpath="{.items[0].metadata.name}")
```

```
found a pod to schedule: default / sleep-5c78857b77-jdfnp
2019/03/01 05:58:29 ip-172-20-43-133.us-west-1.compute.internal
2019/03/01 05:58:29 ip-172-20-61-109.us-west-1.compute.internal
2019/03/01 05:58:29 calculated priorities: map[ip-172-20-43-133.us-west-1.compute.internal:76 ip-172-20-61-109.us-west-1.compute.internal:19]
Placed pod [default/sleep-5c78857b77-jdfnp] on ip-172-20-43-133.us-west-1.compute.internal
```

At last, check the `events` with `kubectl`. You should see that the sleep pods were scheduled by the `random-scheduler`.

```
kubectl get events | grep Scheduled
```

output:
```
4m          4m           1       sleep-5c78857b77-jdfnp-tqdrh                         Pod                                              Normal   Scheduled           random-scheduler                                      Placed pod [default/sleep-5c78857b77-jdfnp] on ip-172-20-43-133.us-west-1.compute.internal
```

Congrats! You've successfully deployed a custom scheduler! 

## Cleanup 
Run the following to clean everything up 
```
kubectl delete -f manifests/sleep.yaml
kubectl delete -f manifests/random-scheduler.yaml
kubectl delete -f manifests/rbac.yaml
```

Confirm everything was deleted. 
```
kubectl get pods 
```

## Lab Complete
