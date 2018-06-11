# Advanced Kubernetes Course Site

## lab VMs
[VM Sheet](lab_ip/)
### SSH 
The SSH private key is available in `keys` directory. 

To SSH into the VM please make sure permissions are set correctly on the key

```
chmod 600 /path/to/lab
```

The username for SSH is `ubuntu`

```
ssh -i /path/to/k8s-lab ubuntu@<labvmIP>

```

## Course Content
[Slides](https://bit.ly/adv-k8s-content)  

## Labs

### Day 1
Lab 1: [Install Kubernetes](labs/01-install-k8s/)  
Lab 2: [Advanced scheduling](labs/02-affinity/)  
Lab 3: [Custom scheduler](labs/03-scheduler/)  
Lab 4: [Kubelet](labs/04-kubelet/)  
Lab 5: [Create ConfigMap](labs/05-configmap/)  

### Day 2
Lab 6: [Networking & health checks](labs/06-networking/)  
Lab 7: [Services](labs/07-services/)  
Lab 8: [DNS](labs/08-dns/)  
Lab 9: [Secrets](labs/09-secrets/)  
Lab 10: [Init containers](labs/10-init/)  

### Optional lab (Minikube)
Minikube is an all-in-one deployment of a Kubernetes cluster which can be run on your local machine.  The following labs provide steps for installing it on Mac, Windows and Ubuntu. 

Lab 11: [Mac](labs/11-mini-mac/)
[Ubuntu](labs/11-mini-ubuntu/)
[Windows](labs/11-mini-win/)
