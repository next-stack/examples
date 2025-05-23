## GlusterFS


NOTE: GlusterFS in-tree storage driver ( `kubernetes.io/glusterfs`) which was
deprecated in kubernetes 1.25 release is removed entirely  in `v1.26`. Volumes
must be migrated to an alternate storage solution before upgrading to `v1.26`.

[GlusterFS](http://www.gluster.org) is an open source scale-out filesystem.
These examples provide information about how to allow containers use GlusterFS
volumes.

The example assumes that you have already set up a GlusterFS server cluster and
have a working GlusterFS volume ready to use in the containers.

#### Prerequisites

* Set up a GlusterFS server cluster
* Create a GlusterFS volume
* If you are not using hyperkube, you may need to install the GlusterFS client
  package on the Kubernetes nodes
  ([Guide](https://docs.gluster.org/en/latest/Administrator-Guide/Setting-Up-Clients/))

#### Create endpoints

The first step is to create the GlusterFS endpoints definition in Kubernetes.
Here is a snippet of [glusterfs-endpoints.yaml](glusterfs-endpoints.yaml):

```yaml
subsets:
- addresses:
  - ip: 10.240.106.152
  ports:
  - port: 1
- addresses:
  - ip: 10.240.79.157
  ports:
  - port: 1
```

The `subsets` field should be populated with the addresses of the nodes in the
GlusterFS cluster. It is fine to provide any valid value (from 1 to 65535) in
the `port` field.

Create the endpoints:

```sh
$ kubectl create -f examples/volumes/glusterfs/glusterfs-endpoints.yaml
```

You can verify that the endpoints are successfully created by running

```sh
$ kubectl get endpoints
NAME                ENDPOINTS
glusterfs-cluster   10.240.106.152:1,10.240.79.157:1
```

We also need to create a service for these endpoints, so that they will
persist. We will add this service without a selector to tell Kubernetes we want
to add its endpoints manually. You can see
[glusterfs-service.yaml](glusterfs-service.yaml) for details.

Use this command to create the service:

```sh
$ kubectl create -f examples/volumes/glusterfs/glusterfs-service.yaml
```


#### Create a Pod

The following *volume* spec in [glusterfs-pod.yaml](glusterfs-pod.yaml)
illustrates a sample configuration:

```yaml
volumes:
- name: glusterfsvol
  glusterfs:
    endpoints: glusterfs-cluster
    path: kube_vol
    readOnly: true
```

The parameters are explained as the followings.

- **endpoints** is the name of the Endpoints object that represents a Gluster
  cluster configuration. *kubelet* is optimized to avoid mount storm, it will
  randomly pick one from the endpoints to mount. If this host is unresponsive,
  the next Gluster host in the endpoints is automatically selected.
- **path** is the Glusterfs volume name.
- **readOnly** is the boolean that sets the mountpoint readOnly or readWrite.

Create a pod that has a container using Glusterfs volume,

```sh
$ kubectl create -f examples/volumes/glusterfs/glusterfs-pod.yaml
```

You can verify that the pod is running:

```sh
$ kubectl get pods
NAME             READY     STATUS    RESTARTS   AGE
glusterfs        1/1       Running   0          3m
```

You may execute the command `mount` inside the container to see if the
GlusterFS volume is mounted correctly:

```sh
$ kubectl exec glusterfs -- mount | grep gluster
10.240.106.152:kube_vol on /mnt/glusterfs type fuse.glusterfs (rw,relatime,user_id=0,group_id=0,default_permissions,allow_other,max_read=131072)
```

You may also run `docker ps` on the host to see the actual container.

