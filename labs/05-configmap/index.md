# Advanced Kubernetes Lab 05
# ConfigMap Redis cache

## Create ConfigMap for Redis Cache
```
kubectl create configmap example-redis-config --from-file=config/redis-config
```

## Verify ConfigMap looks correct
```
kubectl get configmap example-redis-config -o yaml
```

It should look something like 
```
apiVersion: v1
data:
  redis-config: |
    maxmemory 2mb
    maxmemory-policy allkeys-lru
kind: ConfigMap
metadata:
  creationTimestamp: 2016-03-30T18:14:41Z
  name: example-redis-config
  namespace: default
  resourceVersion: "24686"
  selfLink: /api/v1/namespaces/default/configmaps/example-redis-config
  uid: 460a2b6e-f6a3-11e5-8ae5-42010af00002
```

## Create Redis POD that uses ConfigMap
```
kubectl create -f manifests/redis-pod.yaml
```

## Confirm Redis POD launched with ConfigMap settings
```
kubectl exec -it redis redis-cli
CONFIG GET maxmemory
```

Should see something like this 
```
1) "maxmemory"
2) "2097152"
127.0.0.1:6379> CONFIG GET maxmemory-policy
1) "maxmemory-policy"
2) "allkeys-lru"
```

## Cleanup
```
kubectl delete pod redis
```