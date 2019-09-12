# Advanced Kubernetes Lab 04
# Install Kubelet binary on Container Linux
## SSH into Container Linux instance
`ssh -i </path/to/lab.pem> core@<ip of instance>`

## Download Kubelet serviced file 
```
wget -q --show-progress --https-only --timestamping \
  https://raw.githubusercontent.com/kelseyhightower/standalone-kubelet-tutorial/master/kubelet.service
```

## Move Kubelet service file to proper location 
```
sudo mv kubelet.service /etc/systemd/system/
```

## Start the Kubelet service 
```
sudo systemctl daemon-reload
sudo systemctl enable kubelet
sudo systemctl start kubelet
```

## Verify Kubelet 
```
sudo systemctl status kubelet
```

## Verify no containers are running 
```
docker ps 
```

## Verify no container images are installed
```
docker images 
```

## Create the Kubelet manifests directory 
```
sudo mkdir -p /etc/kubernetes/manifests
```

## Download the `app-v0.1.0.yaml` pod manifest
```
wget -q --show-progress --https-only --timestamping \
  https://raw.githubusercontent.com/kelseyhightower/standalone-kubelet-tutorial/master/pods/app-v0.1.0.yaml
```

## Move manifest file to `/etc/kubernetes/manifests`
```
sudo mv app-v0.1.0.yaml /etc/kubernetes/manifests/app.yaml
```

> Notice the app-v0.1.0.yaml pod manifest is being renamed to app.yaml. This prevents our application from being deployed twice. Each pod must have a unique metadata.name.  

## List installed container images
```
docker images 
```

Now you should see 3 container images. 
```
REPOSITORY                             TAG                 IMAGE ID            CREATED             SIZE
gcr.io/hightowerlabs/app               0.1.0               c7d7002a0776        3 months ago        6.325 MB
gcr.io/hightowerlabs/configurator      0.1.0               164e54187008        3 months ago        2.346 MB
gcr.io/google_containers/pause-amd64   3.0                 99e59f495ffa        20 months ago       746.9 kB
```

## List running containers 
```
docker ps 
```

> You should see three containers running which represent the app pod. Docker does not understand pods so the containers are listed as individual containers following the Kubernetes naming convention.  

Verification 
At this point the app pod is up and running on port 80 in the host namespace. Make a HTTP request to the app pod using the localhost address:

```
curl http://127.0.0.1
```

```
version: 0.1.0
hostname: ip-172-31-22-14.us-west-1.compute.internal
key: 1515022737
```

Wait about 30 seconds and make another HTTP request to the app pod:

```
curl http://127.0.0.1
```

```
version: 0.1.0
hostname: ip-172-31-22-14.us-west-1.compute.internal
key: 1515022977
```

> Notice the `key` field has changed. This configuration setting is being updated by the configurator sidecar container running in the app pod.  
## Testing remote access
The app pod is listening on 0.0.0.0:80 in the host network and is accessible via the external IP of the compute instance

Get the external IP of the instance.
`curl icanhazip.com`

Now in your local terminal run 
`curl http://<EXTERNAL_IP>`

```
version: 0.1.0
hostname: ip-172-31-22-14.us-west-1.compute.internal
key: 1515023097
```

## Update Static Pods
Download the `app-v0.2.0.yaml` pod manifest:
```
wget -q --show-progress --https-only --timestamping \
  https://raw.githubusercontent.com/kelseyhightower/standalone-kubelet-tutorial/master/pods/app-v0.2.0.yaml
```

Move the `app-v0.2.0.yaml` pod manifest to the kubelet manifest directory:
```
sudo mv app-v0.2.0.yaml /etc/kubernetes/manifests/app.yaml
```

> Notice the `app-v0.2.0.yaml` is being renamed to `app.yaml`. This overwrites the current pod manifest and will force the kubelet to upgrade the app pod.  

List the installed container images:
```
docker images
```

```
REPOSITORY                             TAG                 IMAGE ID            CREATED             SIZE
gcr.io/hightowerlabs/app               0.1.0               c7d7002a0776        3 months ago        6.325 MB
gcr.io/hightowerlabs/app               0.2.0               3028d4a68eb1        3 months ago        6.325 MB
gcr.io/hightowerlabs/configurator      0.1.0               164e54187008        3 months ago        2.346 MB
gcr.io/google_containers/pause-amd64   3.0                 99e59f495ffa        20 months ago       746.9 kB
```

> Notice the gcr.io/hightowerlabs/app:0.2.0 image has been added to the local repository.  

## Verification of upgrade 
At this point app version 0.2.0 is up and running. Make a HTTP request to the app pod using the localhost address:
```
curl http://127.0.0.1
```

The version is `0.2.0`
```
version: 0.2.0
hostname: ip-172-31-22-14.us-west-1.compute.internal
key: 1515026740
```
