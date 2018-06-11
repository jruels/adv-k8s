# Advanced Kubernetes Course Site

## lab VMs
<table>
<tr><th>vm number</th><th>public ip</th></tr>
<tr><td>lab1</td> <td>34.239.183.99</td></tr>
<tr><td>lab2</td> <td>34.227.68.89</td></tr>
<tr><td>lab3</td> <td>34.207.238.200</td></tr>
<tr><td>lab4</td> <td>54.236.13.162</td></tr>
<tr><td>lab5</td> <td>34.230.24.65</td></tr>
<tr><td>lab6</td> <td>54.90.225.67</td></tr>
<tr><td>lab7</td> <td>35.173.179.86</td></tr>
<tr><td>lab8</td> <td>35.153.139.54</td></tr>
</table>

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
Lab 11: [Ubuntu](labs/11-mini-ubuntu/)
Lab 11: [Windows](labs/11-mini-win/)
