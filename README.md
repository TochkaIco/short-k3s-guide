# K3S
Okay, so this is a short guide of k3s (mostly for me to have certain commands written down somewhere). [K3S](https://docs.k3s.io/) is lightweight kubernetes. There are many-many different ways of setting your cluster, though.

## What is a cluster?
A cluster is a group of similar things, people, or units gathered or occurring closely together. In the context of kubernetes, a cluster is a group of nodes (individual servers) that are connected with each other. Kubernetes runs your application containers in so called pods, a pod can have one or more containers.

## Kubernetes HA
A term you have likely heard by now is high-availability (HA), it means that the system should handle one **or more** pods/nodes going down without the service going down with them. The main idea of HA is that you have multiple instances (commonly called replicas) of your app containers running at the same time, preferably on separate nodes.

For a complete high-availability kubernetes cluster you would want to have 3 or more nodes. The idea is that you always have at least 2 nodes serving your applications, so in case one of the nodes goes down, you still have that extra layer of protection. That said, even with a 2 node cluster you can still enjoy the advantages of kubernetes HA, just a tiny bit less :)

## Storage
### PersistentVolumeClaims (PVC)
You might have seen the "pvc" marking before, but you probably wondered what it meant. As the name suggests, a persistent volume claim is the kubernetes form of reserving a part of the disk (persistent storage) that fits the required disk space. Those bits of space that end up being reserved are therefore reffered to as pv.

By default, kubernetes does not allow you to resize a pvc after creation, unless you set it's StorageClass permits expansion (``allowVolumeExpansion: true``) and the storage driver supports it.

If you have to recreate a pvc but don't want to lose all it's data, you can create a new pvc and copy all the data with this temporary pod:
```
kubectl -n <namespace> run data-mover --image=alpine:latest --command -- sh -c "apk add --no-cache rsync && rsync -aHAX --progress /old/ /new/ && sleep 30" \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "data-mover",
        "image": "alpine:latest",
        "command": ["sh", "-c", "apk add --no-cache rsync && rsync -aHAX --progress /old/ /new/ && sleep 30"],
        "volumeMounts": [
          {"name": "old-vol", "mountPath": "/old"},
          {"name": "new-vol", "mountPath": "/new"}
        ]
      }],
      "volumes": [
        {"name": "old-vol", "persistentVolumeClaim": {"claimName": "<pvc-name>"}},
        {"name": "new-vol", "persistentVolumeClaim": {"claimName": "<pvc-name>"}}
      ]
    }
  }'
```
After the process is finished, you can delete the data-mover pod by running:
```
kubectl -n <namespace> delete pod data-mover
```

### Storage in a cluster
Since the idea is that your system should survive a node going down, even if you have a replica of the app running on the surviving node, this doesn't necessarily save you (unless you run a container that doesn't need persistant storage). Say you have nodes A and B, both nodes run replicas of your app, the persistent volume sits on node A. Something happens and node A goes down, the app continues running thanks to node B, but it no longer has access to it's storage and will likely crash.

This is where distributed block storage providers come in. I will take [Longhorn](https://longhorn.io/) as an example. Longhorn can pratically combine the disk of all your nodes into one, the data will still be stored on your individual disks, but the engine will manage that for you, on a higher level of abstraction you can look at it as having one bigger bucket of storage. You get the benefits of:
- having more usable storage
- having disaster recovery from a node going completely offline (longhorn puts replicas of the pv on both nodes)

## Namespaces
Namespaces are used in kubernetes to structure all the services related to an app together. You can also set resource limits per namespace. Most ``kubectl`` commands you will be using require that you specify the namespace the manipulated object lives in.

## Manifests
Kubernetes manifests define all of the services a developer may need - a pvc, deployment of multiple containers, ingress service, secrets (environment variables like API keys) and config maps (non-sensitive environment variables), all saved in one or multiple yaml files.
Here is an example of how a pvc manifest can look like:
```
apiVersion: apps/v1
kind: PersistentVolumeClaim
metadata:
  name: forgejo-lh-data
  namespace: forgejo
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 35Gi
```
Once you have defined a manifest and want to apply it, you can do so with this command - ``kubectl apply -f <file-name.yaml>``.

## Safely dissconnecting nodes
If you need to update&reboot one of the nodes on your cluster, etc. you should **not** just run ``sudo apt upgrade -y; sudo reboot``, this can introduce an unnecessary layer of potential issues.

Instead, you should first tell kubernetes to stop scheduling new pods to the node. To do that, run this:
```
kubectl cordon <node-name>
```
After this, you should also drain the node of the pods that are already running on it. You can do that with:
```
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```
Once this is done, you can safely do maintenance!

Run this command to re-activate the node after the maintenance is done:
```
kubectl uncordon <node-name>
```
