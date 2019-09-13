# Kubernetes Advanced Course Lab 06
## Use container probes for health checks 
Every Pod resource has a `restartPolicy` that applies to all of the containers for the Pod. By default, the `restartPolicy` is `Always`, which means that the `kubelet` on the node hosting that pod will automatically restart the pod’s containers if they exit or fail. This is the correct policy for applications that are not expected to terminate, like web applications or services like the `simpleservice`.

It’s sometimes the case that long-running applications can end up in a problematic state without crashing, and the `kubelet` will fail to recognize the need to intervene. For this reason, Kubernetes allows you to define different types of probe operations against containers, which the `kubelet` will execute periodically to assess container state.

## Create Pod with liveness-exec probe
Open `simple_liveness.yaml` in an editor and review:
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

To see this in operation, create the resource from `simple_liveness.yaml`
```
kubectl apply -f manifests/simple_liveness.yaml
```

Then use `kubectl` get with the -w option to list the pod details and then watch for changes. This will enable us to see updates to the status of running Pods as they are witnessed by Kubernetes.
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
kubectl apply -f manifests/tcp-liveness.yaml
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

## Define readiness-exec probe
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

## Combine readinesss and liveness Probes
Readiness and liveness probes can be used in parallel for the same container. Using both can ensure that traffic does not reach a container that is not ready for it, and that containers are restarted when they fail.

To demonstrate this I have created a manifest which has both readiness and liveness probes. 

Look at `manifests/helloworld-liveness-readiness.yml`

The important part is: 
```
spec:
      containers:
      - name: k8s-demo
        image: wardviaene/k8s-demo
        ports:
        - name: nodejs-port
          containerPort: 3000
        livenessProbe:
          httpGet:
            path: /
            port: nodejs-port
          initialDelaySeconds: 15
          timeoutSeconds: 30
        readinessProbe:
          httpGet:
            path: /
            port: nodejs-port
          initialDelaySeconds: 15
          timeoutSeconds: 30
```

As you can see we are doing an `httpGet` request on the `nodejs-port` for our `livenessProbe` and then we are also doing a `readinessProbe` check to confirm the container comes online and is available to serve our application. 

Now let's see the different between a deployment with just a `livenessProbe` and a deployment with both `livenessProbe and readinessProbe`

Start by deploying `helloworld-healthcheck.yml` and monitor the status
```
kubectl apply -f manifests/helloworld-healthcheck.yml && watch kubectl get pods
```

You will see that the Pods are scheduled and immediately go into a `Ready` state which means Kubernetes will start sending traffic to them. 

As we discussed however there are many times where your application takes a bit longer to come online and we don't want our customers being directed to Pods that do not have a fully functioning application. 

To avoid this we also add a `readinessProbe` 

Deploy `helloworld-liveness-readiness.yml` and monitor the status
```
kubectl apply -f manifests/helloworld-liveness-readiness.yml && watch kubectl get pods 
```

Now you will see that the Pods are scheduled, but they do not enter a 	`Ready` state until the nodejs server is responding to `GET` requests on `/`

## Lab Complete



