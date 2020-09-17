#**Deploy OCS On Cluster**

Now that we had added an additional worker node to our lab cluster environment we can deploy OpenShift Containerized Storage on top of the cluster.

First we need to attach a 100GB disk to each of our worker nodes in the lab environment.  Thankfully we have a little script to do this for us on the provisioning node.

~~~bash

~~~

We can validate that each node has the extra disk by using the debug container on the worker node:

~~~bash

~~~

Once inside the debug container we can look at the block devices available:

~~~bash

~~~

We can repeat the above steps in the lab for each worker node.

Now that we know the worker nodes have their disk we can proceed.  Before installing OCS we should first install the local-storage operator which we can configure the local disks on the worker nodes which in turn can be consumed by OCS as OSD devices.




Now that we have the local storage operator installed lets make a storage definition file that will use the disk device in each node:

~~~bash
[cloud-user@provision scripts]$ cat << EOF > ~/local-storage.yaml
apiVersion: local.storage.openshift.io/v1
kind: LocalVolume
metadata:
  name: local-block
  namespace: local-storage
spec:
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
        - key: cluster.ocs.openshift.io/openshift-storage
          operator: In
          values:
          - ""
  storageClassDevices:
    - storageClassName: localblock
      volumeMode: Block
      devicePaths:
        - /dev/vdb
EOF
~~~

Let's take a look at the file that it created:

~~~bash
[cloud-user@provision scripts]$ cat ~/local-storage.yaml
apiVersion: local.storage.openshift.io/v1
kind: LocalVolume
metadata:
  name: local-block
  namespace: local-storage
spec:
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
        - key: cluster.ocs.openshift.io/openshift-storage
          operator: In
          values:
          - ""
  storageClassDevices:
    - storageClassName: localblock
      volumeMode: Block
      devicePaths:
        - /dev/vdb
~~~

You'll see that this is set to create local volume on every host from the block device vdb where the selector key matches cluster.ocs.openshift.io/openshift-storage
