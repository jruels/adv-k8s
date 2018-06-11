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
