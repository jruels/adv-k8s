# Kubernetes Advanced Course Lab 06
# Pod Networking

## Review Pod manifest
Open `simple_pod.yaml` in a text editor and review the configuration. 
```
vim manifests/simple_pod.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
    name: simple-api
    labels:
      lab: podlab
spec:
    containers:
    - name: simpleservice
      image: mhausenblas/simpleservice:0.5.0
      ports:
      - containerPort: 9876
      env:
      - name: DEMO_GREETING
        value: "Hello from the environment!!"
```

NOTE: The `env` section.  

## Create Pod from manifest
```
kubectl apply -f manifests/simple_pod.yaml
```

## Confirm Pod is running 
```
kubectl get pods simple-api -o wide 
```

```
NAME         READY     STATUS    RESTARTS   AGE       IP          NODE
simple-api   1/1       Running   0          29s       10.44.0.1   ip-10-0-100-70
```
Make a note of `IP` for next step. 

## Access web service 
Because we have not yet exposed the `simple-api` service externally it is only accessible through the container network.  So we are going to use a jump container to log in and run commands on the container network.

## Create jump container 
```
kubectl run -i -t --rm jumpserver --image=satoms/jumpserver:v1 --restart=Never
```

**NOTE: The --restart policy flag on the kubectl run command determines whether the command will start a Deployment (--restart=Always), a Job (--restart=OnFailure), or a bare pod (--restart=Never). The -i and -t flags function similarly to Docker flags to instantiate an interactive TTY in the foreground for the pod container. The --rm flag ensures that the pod resources are deleted when the pod container exits.**

Now let’s `curl` the `simple-api` Pod on the `clusterIP`  
```
curl http://<CLUSTER-IP>:9876/info
```

Returns something similar to:
```
{"host": "10.44.0.1:9876", "version": "0.5.0", "from": "10.36.0.2"}/ #
```

We can also call the /**env** endpoint to request the service echo all of its runtime environment variables:

```
curl http://<CLUSTER-IP>:9876/env
```

You’ll get output like below: 
```
{"version": "0.5.0", "env": "{'LANG': 'C.UTF-8', 'KUBERNETES_PORT_443_TCP_PROTO': 'tcp', 'KUBERNETES_PORT_443_TCP': 'tcp://10.96.0.1:443', 'PYTHON_VERSION': '2.7.13', 'PYTHON_PIP_VERSION': '9.0.1', 'KUBERNETES_SERVICE_HOST': '10.96.0.1', 'HOSTNAME': 'simple-api', 'KUBERNETES_SERVICE_PORT_HTTPS': '443', 'DEMO_GREETING': 'Hello from the environment!!', 'REFRESHED_AT': '2017-04-24T13:50', 'GPG_KEY': 'C01E1CAD5EA2C4F0B8E3571504C367C218ADD4FF', 'KUBERNETES_PORT_443_TCP_ADDR': '10.96.0.1', 'KUBERNETES_PORT': 'tcp://10.96.0.1:443', 'PATH': '/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin', 'KUBERNETES_PORT_443_TCP_PORT': '443', 'HOME': '/root', 'KUBERNETES_SERVICE_PORT': '443'}"}/ #
```

Notice the environment  set in the POD manifest has been injected into the environment.
```
'DEMO_GREETING': 'Hello from the environment!!'
```

## Expose `simple-api` service externally 
Now if we want to access this service without using a jump container we have to expose the service and map the container port to the Kubernetes node port. 

```
kubectl expose pod simple-api --port=4000 --target-port=9876 --name=simple-api
```

## Get service connection info
```
kubectl get svc simple-api
```

You’ll see the `ClusterIP` displayed
```
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
simple-api   ClusterIP   10.102.115.255   <none>        4000/TCP   9m
```

## Connect to service on `ClusterIP`
Now that we have the service information we can connect using `curl` to the `ClusterIP`
```
curl http://10.102.115.255:4000/info
```

## Cleanup 
```
kubectl delete pod simple-api
```

## Use container probes for health checks 
Every Pod resource has a restartPolicy that applies to all of the containers for the pod. By default, the `restartPolicy` is `Always`, which means that the `kubelet` on the node hosting that pod will automatically restart the pod’s containers if they exit or fail. This is the correct policy for applications that are not expected to terminate, like web applications or services like the `simpleservice`.

It’s sometimes the case that long-running applications can end up in a problematic state without crashing, and the `kubelet` will fail to recognize the need to intervene. For this reason, Kubernetes allows you to define different types of probe operations against containers, which the `kubelet` will execute periodically to assess container state.

## Create Pod with liveness probe
Open `simple_liveness.yaml` in an editor and update to look like below:
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

To see this in operation, create the resource from `simple_liveness.yaml`, as before:
```
kubectl apply -f manifests/simple_liveness.yaml
```

Then use kubectl get with the -w option to list the pod details and then watch for changes. This will enable us to see updates to the status of running Pods as they are witnessed by Kubernetes.
```
kubectl get pods liveness-exec -w -o wide
```

In the configuration file, you can see that the Pod has a single Container. The `periodSeconds` field specifies that the kubelet should perform a liveness probe every 5 seconds. The `initialDelaySeconds` field tells the kubelet that it should wait 5 second before performing the first probe. To perform a probe, the kubelet executes the command `cat /tmp/healthy` in the Container. If the command succeeds, it returns 0, and the kubelet considers the Container to be alive and healthy. If the command returns a non-zero value, the kubelet kills the Container and restarts it.

When the Container starts, it executes this command:
```
/bin/sh -c "touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600"
```

For the first 30 seconds of the Container’s life, there is a `/tmp/healthy` file. So during the first 30 seconds, the command `cat /tmp/healthy` returns a success code. After 30 seconds, `cat /tmp/healthy` returns a failure code.

Within 30 seconds, view the Pod events:
```
kubectl describe pod liveness-exec
```

You should see that everything is running and passing the liveness probe. 
```
FirstSeen    LastSeen    Count   From            SubobjectPath           Type        Reason      Message
--------- --------    -----   ----            -------------           --------    ------      -------
24s       24s     1   {default-scheduler }                    Normal      Scheduled   Successfully assigned liveness-exec to worker0
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulling     pulling image "k8s.gcr.io/busybox"
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulled      Successfully pulled image "k8s.gcr.io/busybox"
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Created     Created container with docker id 86849c15382e; Security:[seccomp=unconfined]
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Started     Started container with docker id 86849c15382e
```

After 35 seconds, view the Pod events again:
```
kubectl describe pod liveness-exec
```

At the bottom of the output, there are messages indicating that the liveness probes have failed, and the containers have been killed and recreated.
```
Events:
  Type     Reason                 Age               From                     Message
  ----     ------                 ----              ----                     -------
  Normal   Scheduled              50s               default-scheduler        Successfully assigned liveness-exec to ip-10-0-100-70
  Normal   SuccessfulMountVolume  50s               kubelet, ip-10-0-100-70  MountVolume.SetUp succeeded for volume "default-token-9lx6d"
  Normal   Pulling                49s               kubelet, ip-10-0-100-70  pulling image "k8s.gcr.io/busybox"
  Normal   Pulled                 49s               kubelet, ip-10-0-100-70  Successfully pulled image "k8s.gcr.io/busybox"
  Normal   Created                49s               kubelet, ip-10-0-100-70  Created container
  Normal   Started                48s               kubelet, ip-10-0-100-70  Started container
  Warning  Unhealthy              6s (x3 over 16s)  kubelet, ip-10-0-100-70  Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
```

Wait another 30 seconds, and verify that the Container has been restarted:
```
kubectl get pod liveness-exec
```

The output shows that `RESTARTS` has been incremented:
```
NAME            READY     STATUS    RESTARTS   AGE
liveness-exec   1/1       Running   1          2m
```

## Define a Liveness HTTP request 
Another kind of liveness probe uses an HTTP GET request. 

Here is the POD manifest file used in this lab: 
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/liveness
    args:
    - /server
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```

In the configuration file, you can see that the Pod has a single Container. The `periodSeconds` field specifies that the kubelet should perform a liveness probe every 3 seconds. The `initialDelaySeconds` field tells the kubelet that it should wait 3 seconds before performing the first probe. To perform a probe, the kubelet sends an HTTP GET request to the server that is running in the Container and listening on port 8080. If the handler for the server’s `/healthz` path returns a success code, the kubelet considers the Container to be alive and healthy. If the handler returns a failure code, the kubelet kills the Container and restarts it.

Any code greater than or equal to 200 and less than 400 indicates success. Any other code indicates failure.

For the first 10 seconds that the Container is alive, the `/healthz` handler returns a status of `200`. After that, the handler returns a status of `500`.

Here is the code for the server written in GOLANG
```
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    duration := time.Now().Sub(started)
    if duration.Seconds() > 10 {
        w.WriteHeader(500)
        w.Write([]byte(fmt.Sprintf("error: %v", duration.Seconds())))
    } else {
        w.WriteHeader(200)
        w.Write([]byte("ok"))
    }
})
```

The kubelet starts performing health checks 3 seconds after the Container starts. So the first couple of health checks will succeed. But after 10 seconds, the health checks will fail, and the kubelet will kill and restart the Container.

To try the HTTP liveness check, create a Pod:
```
kubectl create -f manifests/http-liveness.yaml
```

After 10 seconds, view Pod events to verify that liveness probes have failed and the Container has been restarted:
```
kubectl describe pod liveness-http
```

## Define a TCP liveness probe
A third type of liveness probe uses a TCP Socket. With this configuration, the kubelet will attempt to open a socket to your container on the specified port. If it can establish a connection, the container is considered healthy, if it can’t it is considered a failure.
```
apiVersion: v1
kind: Pod
metadata:
  name: goproxy
  labels:
    app: goproxy
spec:
  containers:
  - name: goproxy
    image: k8s.gcr.io/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

As you can see, configuration for a TCP check is quite similar to an HTTP check. This example uses both readiness and liveness probes. The kubelet will send the first readiness probe 5 seconds after the container starts. This will attempt to connect to the `goproxy` container on port 8080. If the probe succeeds, the pod will be marked as ready. The kubelet will continue to run this check every 10 seconds.

In addition to the readiness probe, this configuration includes a liveness probe. The kubelet will run the first liveness probe 15 seconds after the container starts. Just like the readiness probe, this will attempt to connect to the `goproxy` container on port 8080. If the liveness probe fails, the container will be restarted.

Go ahead and create it:
```
kubectl apply -f tcp-liveness.yaml
```


## Use a named port
You can use a named container port for HTTP or TCP liveness checks:
```
ports:
- name: liveness-port
  containerPort: 8080
  hostPort: 8080

livenessProbe:
  httpGet:
    path: /healthz
    port: liveness-port
```

## Define readiness probes
Sometimes, applications are temporarily unable to serve traffic. For example, an application might need to load large data or configuration files during startup. In such cases, you don’t want to kill the application, but you don’t want to send it requests either. Kubernetes provides readiness probes to detect and mitigate these situations. A pod with containers reporting that they are not ready does not receive traffic through Kubernetes Services.

Readiness probes are configured similarly to liveness probes. The only difference is that you use the `readinessProbe` field instead of the `livenessProbe` field.
```
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

Configuration for HTTP and TCP readiness probes also remains identical to liveness probes.

Readiness and liveness probes can be used in parallel for the same container. Using both can ensure that traffic does not reach a container that is not ready for it, and that containers are restarted when they fail.



