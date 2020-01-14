# Kubernetes Advanced Course Lab08
# DNS and Services
Accessing a service over DNS is much easier than needing to remember its IP address.  In this lab we are going to deploy a simple 2 tier application Wordpress and confirm the tiers can talk to each other using DNS resolution. 

In this example, you will use an EmptyDir volume type that doesn’t persistent, but that allows you to perform this tutorial without having to have any specific volume providers.

Let’s start by creating a new namespace 
```
kubectl create ns wordpress-example
```

Next let’s create our MariaDB deployment resource:
```
kubectl create -f manifests/mariadb-deployment.yaml
```

Check that the database is running:
```
kubectl -n wordpress-example get pod
```

You should see something like: 
```
NAME                    READY     STATUS    RESTARTS   AGE
mysql-7b8c5cf67-8lfqb   1/1       Running   0          19s
```

To allow any other container to speak to our database pod we need to expose it to a service resource:
```
kubectl create -f manifests/mysql_svc.yaml
```

As you can see you have exposed our MariaDB service (port 3306) behind the name mysql on the namespace wordpress-example .

Therefore from any other pod you may directly talk to mysql if you are in the same namespace or mysql.wordpress-example.svc.cluster.local from any other namespace.

It’s now time to create our Wordpress container and configure it to talk to our MariaDB pod:
```
kubectl create -f manifests/wordpress-deployment.yaml
```

Now check whether the POD is running correctly. 
```
kubectl -n wordpress-example get pod
```

Should see something like this: 
```
mysql-3060656199-z7c8z                     1/1       Running   0          
wordpress-3190343080-djzjb                 1/1       Running   0
```

To reach the Wordpress website you need to expose the Wordpress service. In this instance a ClusterIP isn’t sufficient because you need to be able to contact it from the exterior of the cluster.

We’ll use `NodePort`
```
kubectl create -f manifests/wordpress_svc.yaml
```

Now let’s figure out the service port
```
kubectl -n wordpress-example get svc 
```

We can now confirm everything is working by opening in a browser 
```
http://<worker-IP>:31499
```

Fill out with following details: 
* Database name : `wordpress` as defined in the DB’s yaml file
* User name : `root`
* Password : `aStr0ngPassW0rd` as defined in yaml too
* Database host : `mysql` or `mysql.wordpress-example.svc.cluster.local`

As you can see, talking to your DB from your frontend pod is not difficult due to the creation of our mysql service. Without this you would have had to find the actual container IP, and then change it if the pod were recreated due to an update or outage.

## Cleanup
```
kubectl delete ns wordpress-example
```
