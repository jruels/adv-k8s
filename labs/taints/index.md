# Taint and Tolerations Lab
Node affinity  is a property of pods that attracts them to a set of nodes (either as a preference or a hard requirement). Taints are the opposite – they allow a node to repel a set of pods. 

Taints and tolerations work together to ensure that pods are not scheduled onto inappropriate nodes. One or more taints are applied to a node; this marks that the node should not accept any pods that do not tolerate the taints. Tolerations are applied to pods, and allow (but do not require) the pods to schedule onto nodes with matching taints.

You add a taint to a node using `kubectl taint`. For example: 
```
kubectl taint nodes node1 key=value:NoSchedule
```
places a taint on node `node1` The taint has key `key`, value `value`, and taint effect `NoSchedule`.
This means that no pod will be able to schedule onto node1 unless it has a matching toleration. 

You specify a toleration for a pod in the `PodSpec`. Both of the following tolerations “match” the taint created by the `kubectl taint` line above, and thus a pod with either toleration would be able to schedule onto `node1`:

```
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

```
tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"
```

NOTE: An empty `key` with operator `Exists` matches all keys ,values and effects which means it will tolerate everything. 
```
tolerations:
- operator: "Exists"
```

An empty effect matches all effects with a key `key`
```
tolerations:
- key: "key"
  operator: "Exists"
```

## Apply taint to node 
In this lab we will be applying a taint and then showing how only PODs with an exception are allowed to be scheduled on that node. 

## Get nodes 
`kubectl get nodes`

```
NAME              STATUS    ROLES     AGE       VERSION
ip-10-0-100-102   Ready     <none>    1d        v1.9.0
ip-10-0-100-134   Ready     master    1d        v1.9.0
ip-10-0-100-70    Ready     <none>    1d        v1.9.0
```

Apply a taint to the top node. 
```
kubectl taint nodes <ip-10-0-100-102> dedicated=lab:NoSchedule
```

## Deploy nginx without toleration

```
apiVersion: apps/v1beta2 
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3
  template: 
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

```
kubectl apply -f manifests/nginx_deployment.yaml
```

You’ll notice it does not get scheduled to the node we tainted above. 

## Update YAML to allow scheduling on tainted node. 
```
apiVersion: apps/v1beta2 
kind: Deployment
metadata:
  name: nginx-taint
spec:
  selector:
    matchLabels:
      app: nginx-taint
  replicas: 3
  template: 
    metadata:
      labels:
        app: nginx-taint
    spec:
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "lab"
        effect: "NoSchedule"
      containers:
      - name: nginx-taint
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

```
kubectl apply -f manifests/nginx_taint.yaml
```

## Confirm nodes are scheduled as expected
Now confirm only PODs in deployment  `nginx-taint` are running on tainted node. 
```
kubectl get pods -o wide 
``` 

You should see that PODs in deployment `nginx-deployment` are NOT scheduled on tainted node. 

```
NAME                                READY     STATUS    RESTARTS   AGE       IP          NODE
nginx-deployment-6c54bd5869-gst9p   1/1       Running   0          10m       10.44.0.2   ip-10-0-100-70
nginx-deployment-6c54bd5869-nq7kt   1/1       Running   0          10m       10.44.0.3   ip-10-0-100-70
nginx-deployment-6c54bd5869-p48s2   1/1       Running   0          10m       10.44.0.1   ip-10-0-100-70
nginx-taint-9764dfc98-6vrwl         1/1       Running   0          8m        10.44.0.4   ip-10-0-100-70
nginx-taint-9764dfc98-gjv4c         1/1       Running   0          8m        10.36.0.2   ip-10-0-100-102
nginx-taint-9764dfc98-szmrm         1/1       Running   0          8m        10.36.0.1   ip-10-0-100-102
```

## Remove taint
```
kubectl taint nodes <ip-10-0-100-102> dedicated-
```

## Cleanup 
```
kubectl delete deployment nginx-deployment nginx-taint
```


# Job and Cronjob lab
You can use a Job to run batch processes, ETL jobs, ad-hoc operations, etc. It starts off a Pod and lets it run to completion. This is quite different from other Pod controllers such a Deployment or ReplicaSet. You can also use CronJobs to schedule jobs periodically. 


## Here is what a typical Job manifest looks like:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job1
spec:
  template:
    spec:
      containers:
        - name: job
          image: busybox
          args:
            - /bin/sh
            - -c
            - date; echo sleeping....; sleep 90s; echo exiting...; date
      restartPolicy: Never
```

This Job will simply start a busybox container that simply executes a bunch of shell commands. Let's create this Job and investigate what's going on
```
kubectl apply -f manifests/job1.yaml
```

Check the `Job` and its associated `pod`.
```
kubectl get job job1

NAME   COMPLETIONS   DURATION   AGE
job1   0/1           8s         8s
```

You should see a pod in Running state, for e.g.:
```
kubectl get pod -l=job-name=job1
job1-bptmd 1/1  Running
```


If you check the Pod logs, you should see something similar to this:
```
kubectl logs <pod_name>

Thu Jan  9 10:10:35 UTC 2020
sleeping....
```

Check the job again after ~90s.
```
kubectl get job/job1

NAME   COMPLETIONS   DURATION   AGE
job1   1/1           95s        102s
```

The Job ran for a little over 90 seconds and `COMPLETIONS` reflects that one pod completed successfully. This will reflect in the pod logs as well.

```
kubectl logs <pod_name>
```

```
Thu Jan  9 10:10:05 UTC 2020
sleeping....
exiting...
Thu Jan  9 10:11:35 UTC 2020
```

Also, the pod status should change to `Completed`.

```
kubectl get pod -l=job-name=job1

job1-bptmd 0/1  Completed
```

A Job can be restarted by Kubernetes if the container fails, that cannot happen with an isolated pod. In addition to this, there are many other capabilities that a Job Controller provides, which we will explore going forward.

To delete this Job, simply run 

```
kubectl delete job/job1
```

### Enforcing a time limit
For e.g., you are running a batch job and it takes too long to finish due to some reason. This might be undesirable. You can limit the time for which a Job can continue to run by setting the `activeDeadlineSeconds` attribute in the spec.

Here is an example:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job2
spec:
  activeDeadlineSeconds: 5
  template:
    spec:
      containers:
        - name: job
          image: busybox
          args:
            - /bin/sh
            - -c
            - date; echo sleeping....; sleep 10s; echo exiting...; date
      restartPolicy: Never
```

Notice that the `activeDeadlineSeconds` has been set to 5 seconds while the container process has been designated to run for 10 seconds.

Create the Job, wait for a few seconds (~10 seconds) and check the Job.

```
kubectl apply -f manifests/job2.yaml
```

Check the status:
```
kubectl get job/job2 -o yaml
```

Scroll down to check the `status` field and you will see that the Job is in a `Failed` state due to `DeadlineExceeded`.

```
status:
  conditions:
  - lastProbeTime: "2020-01-09T10:57:13Z"
    lastTransitionTime: "2020-01-09T10:57:13Z"
    message: Job was active longer than specified deadline
    reason: DeadlineExceeded
    status: "True"
    type: Failed
```

To delete the job, simply run 
```
kubectl delete job/job2
```

### Handling failures
What if there are issues due to container failure (process exited) or pod failure? Let's try this out by simulating a failure.

In this Job, the container prints the `date`, sleeps for 5 seconds, and exits with a status 1 to simulate failure.


```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job3
spec:
  backoffLimit: 2
  template:
    spec:
      containers:
        - name: job
          image: busybox
          args:
            - /bin/sh
            - -c
            - date; echo sleeping....; sleep 5s; exit 1;
      restartPolicy: OnFailure
```

Notice that the `restartPolicy: OnFailure` is different compared to the previous example where it was set to Never. We will come back to this in a moment.


Create the Job and keep an eye on a specific Pod for this job.

```
kubectl apply -f manifests/job3.yaml
```

Check the jobs status: 
```
kubectl get pod -l=job-name=job3 -w
```


You should see something similar to below:
```
NAME                                     READY   STATUS              RESTARTS   AGE
job3-qgv4b                               0/1     ContainerCreating   0          4s
job3-qgv4b                               1/1     Running             0          6s
job3-qgv4b                               0/1     Error               0          12s
job3-qgv4b                               1/1     Running             1          17s
job3-qgv4b                               0/1     Error               1          22s
job3-qgv4b                               0/1     CrashLoopBackOff    1          34s
job3-qgv4b                               1/1     Running             2          40s
job3-qgv4b                               1/1     Terminating         2          40s
job3-qgv4b                               0/1     Terminating         2          45s
job3-qgv4b                               0/1     Terminating         2          51s
```

Notice how the pod status transitions.

* It starts off by pulling and running the container.
* It transitions to Error state since it exits with status 1 (after sleeping for 5 seconds).
* It goes back to Running status again (notice that the RESTARTS count is now 1).
* As expected, it goes into `Error` state again and is restarted once more - `RESTARTS` count is now 2.
* Finally, it’s `terminated`.


Kubernetes (the Job Controller to be specific) restarted the container for us because we specified `restartPolicy: OnFailure`.

But there might be a situation where this might continue indefinitely, so we put a limit on this using `backoffLimit: 2` which will ensure that Kubernetes re-tries only twice before marking this Job as `Failed`.

Note that this was an example of the container being restarted. the Job controller can also create a new pod in case of a pod failure.
If you check the Job status...

```
kubectl get job/job3 -o yaml
```

… you will see that it has Failed due to `BackoffLimitExceeded`.

```
status:
  conditions:
  - lastProbeTime: "2020-01-09T11:16:24Z"
    lastTransitionTime: "2020-01-09T11:16:24Z"
    message: Job has reached the specified backoff limit
    reason: BackoffLimitExceeded
    status: "True"
    type: Failed
```

`restartPolicy` of `Never` means that a failure will not restart the container or create a new pod when things go wrong. Also, the default limit for `backoffLimit` is 6.

To delete this job, just run 
```
kubectl delete job/job3
```


### Multiple pods
There are requirements where you might want the Job to spin up more than one pod to get things done.

For e.g., consider a scenario where you are running a batch job to process records from a database — having multiple pods share the load can definitely help.

One way of doing this might be for each pod to run sequentially, record the number of rows processed in an external source (e.g. another DB table) and the other pod can pick up from there.

This can be done by adding the completions property in the Job spec.


```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job4
spec:
  completions: 2
  template:
    spec:
      containers:
        - name: job
          image: busybox
          args:
            - /bin/sh
            - -c
            - date; echo sleeping....; sleep 10s; echo exiting...; date
      restartPolicy: Never
```


Create the Job and keep an eye on how it progresses.

```
kubectl apply -f manifests/job4.yaml
```

Check status: 
```
kubectl get job/job4 -w
```

You should see something similar to this:

```
NAME   COMPLETIONS   DURATION   AGE
job4   0/2           3s         3s
job4   1/2           20s        20s
job4   2/2           37s        37s
```

Since we had set `completions` to two:

* Two Pods were instantiated one after the other (sequentially).
* Job was marked Completed (successful) only after both pods ran to completion. Otherwise, the failure conditions would have applied (as discussed above).

Let’s check the pod logs as well.

```
kubectl get pods -l=job-name=job4
kubectl logs <pod_name>

```

If you see the logs for both the Pods, you will be able to confirm that they started one after the other in a sequence (and each ran for ~10 seconds).

Logs for pod1.

```
hu Jan  9 11:31:57 UTC 2020
sleeping....
exiting...
Thu Jan  9 11:32:07 UTC 2020
```

Logs for pod 2

```
Thu Jan  9 11:32:15 UTC 2020
sleeping....
exiting...
Thu Jan  9 11:32:25 UTC 2020
```

How about running the batch processing in a parallel fashion where all the Pods are instantiated at once (instead of sequentially)?
To handle this case, our processing logic needs to be tuned accordingly since there is co-ordination required amongst the parallel pods in terms of which set of work items to pick and how to update their completion status.
We will not dive into that, but I hope you get the idea in terms of the requirement.
Now, this can be achieved by using parallelism along with completions. Here is an example:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job5
spec:
  completions: 3
  parallelism: 3
  template:
    spec:
      containers:
        - name: job
          image: busybox
          args:
            - /bin/sh
            - -c
            - date; echo sleeping....; sleep 10s; echo exiting...; date
      restartPolicy: Never
```

By using the parallelism attribute, we were able to put a cap on the maximum number of pods that can run at a time. In this case, since parallelism is set to three, it implies that:

* Three Pods will be instantiated all at once.
* Job will be marked Completed (successful) only if all three run to completion. Otherwise, the failure conditions apply (as discussed above).

### Once you're done
You can use `ttlSecondsAfterFinished` to specify the number of seconds after which the Job can be automatically deleted once it is finished (either Completed or Failed). This also removes dependent entities such as Pods spawned by the Job.

# CronJob
A CronJob object allows you to schedule Job execution rather than starting them manually.   

It uses the Cron format to run a job as scheduled. Basically, the CronJob is a higher-level abstraction that embeds within itself a Job template (as seen above) along with a schedule (cron format) and other attributes.
Let’s reviewa simple CronJob that repeats every minute.

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: cronjob1
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cronjob
              image: busybox
              args:
                - /bin/sh
                - -c
                - date; echo sleeping....; sleep 5s; echo exiting...;
          restartPolicy: Never
```

The jobTemplate section is the same as that of a Job. It’s simply embedded within this CronJob spec. It’s the same container that we were using for the Job example.

Create the `CronJob`

```
kubectl apply -f cronjob1.yaml

kubectl get cronjob/cronjob1
```

The output: 
```
NAME       SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob1   */1 * * * *   False     0        <none>          4s
```


Keep track of the Job that this CronJob spawns.

```
kubectl get job -w
```

A new Job is being created every minute and it ran for ~10 seconds as expected. You can also check the logs of the individual pod that the Job created (just like you did with previous examples).

```
kubectl get pod -l=job-name=<job_name>
kubectl logs <pod_name>
```


There are other (optional) CronJob properties in addition to the schedule attribute. Let's look at one of these.

### Concurrentpolicy

It has three possible values — Forbid, Allow, and Replace.

Choose Forbid if you don't want concurrent executions of your Job. When it’s time to trigger a Job as per the schedule and a Job instance is already running, the current iteration is skipped.

If you choose Replace as the concurrency policy, the currently running Job will be stopped and a new Job will be spawned.
Specifying Allow will let multiple Job instances run concurrently.


```yaml
Here is an example:
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: cronjob2
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Allow
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cronjob
              image: busybox
              args:
                - /bin/sh
                - -c
                - date; echo sleeping....; sleep 90s; echo exiting...;
          restartPolicy: Never
```

You can create this CronJob and then track the individual Jobs to observe the behavior.

```
kubectl apply -f manifests/cronjob2.yaml

kubectl get jobs -w
```

Since the schedule is every one minute and the container runs for 90 seconds, you will see multiple Jobs running at the same time. This overlap is possible since we have applied `concurrencyPolicy: Allow`.

You might see something like this:

```
cronjob2-1578573480   0/1                      0s
cronjob2-1578573480   0/1           0s         0s
cronjob2-1578573540   0/1                      0s
cronjob2-1578573540   0/1           0s         0s
cronjob2-1578573480   1/1           95s        95s
```

Notice that job cronjob2-1578573540 was triggered before cronjob2-1578573480 could finish.

The other properties of a CronJob are: 

- Job history: `successfulJobsHistoryLimit` and `failedJobsHistoryLimit` can be used to specify how much history you want to retain for failed and completed Jobs.

- Start deadline specified by startingDeadlineSeconds.
- Suspend specified by suspend

# Drain node 
You can use drain to safely evict all of your pods from a node before you perform maintenance on the node (e.g. kernel upgrade, hardware maintenance, etc.). Safe evictions allow the pod's containers to gracefully terminate and will respect the PodDisruptionBudgets you have specified.

First, identify the name of the node you wish to drain. You can list all of the nodes in your cluster with:
```
kubectl get nodes
```

Next, tell Kubernetes to drain the node:
```
kubectl drain <node name>
```

Once it returns (without giving an error), you can power down the node (or equivalently, if on a cloud platform, delete the virtual machine backing the node). If you leave the node in the cluster during the maintenance operation, you need to run:

```
kubectl uncordon <node name>
```

afterwards to tell Kubernetes that it can resume scheduling new pods onto the node.

## Congrats! 
