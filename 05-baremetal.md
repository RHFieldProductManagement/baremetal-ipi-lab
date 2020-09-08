#**Baremetal Operator**

Now that our cluster is up and running, we can start playing around with it and figure out how all of the baremetal management is configured through the baremetal-operator. The baremetal-operator pods live in the 'openshift-machine-api' namespace:

~~~bash
kni@provisioner$ oc get pods -n openshift-machine-api
NAME                                           READY   STATUS    RESTARTS   AGE
cluster-autoscaler-operator-57c7d59968-hncv5   1/1     Running   0          22m
machine-api-controllers-55f6df9b4c-7gqcf       3/3     Running   0          34m
machine-api-operator-6f64b778d6-z9kmg          1/1     Running   0          33m
metal3-5d5c75dc4b-qzswz                        8/8     Running   0          33m
~~~

Here, the metal3 pod is the pod we're interested in - it's where we house all of the relevant containers, including the baremetal-operator itself:

~~~bash
kni@provisioner$ oc describe pod metal3-5d5c75dc4b-qzswz -n openshift-machine-api
(...)
~~~

As part of the initial bootstrap of the cluster, the OpenShift installer handed control over the underlying baremetal machines to the baremetal-operator running on the cluster and automatically seeded the `BareMetalHost` resources so they can be **adopted** (where Ironic takes ownership of a previously-deployed node), but you'll see that they're in 'error' state:

~~~bash
kni@provisioner$ oc get baremetalhosts -n openshift-machine-api
NAME                 STATUS   PROVISIONING STATUS   CONSUMER       BMC                  HARDWARE PROFILE   ONLINE   ERROR
openshift-master-0   error    registering           kni-master-0   ipmi://192.0.2.221                      true     Failed to get power state for node a4a53134-1690-45c6-93ee-b96c31dbfc54. Error: IPMI call failed: power status.
openshift-master-1   error    registering           kni-master-1   ipmi://192.0.2.222                      true     Failed to get power state for node b2d87e8e-563e-4da4-96f4-99bcb754f7b1. Error: IPMI call failed: power status.
openshift-master-2   error    registering           kni-master-2   ipmi://192.0.2.223                      true     Failed to get power state for node d036a5f7-2424-46f5-8b00-75cc6d9acd1c. Error: IPMI call failed: power status.

~~~

You'll also see that in OpenStack Ironic the nodes are stuck in an '**enroll**' state, not allowing them to progress:

~~~bash
kni@provisioner$ export OS_TOKEN=fake-token
kni@provisioner$ export OS_URL=http://172.22.0.3:6385
kni@provisioner$ openstack baremetal node list
$ openstack baremetal node list
+--------------------------------------+--------------------+--------------------------------------+-------------+--------------------+-------------+
| UUID                                 | Name               | Instance UUID                        | Power State | Provisioning State | Maintenance |
+--------------------------------------+--------------------+--------------------------------------+-------------+--------------------+-------------+
| d036a5f7-2424-46f5-8b00-75cc6d9acd1c | openshift-master-2 | d036a5f7-2424-46f5-8b00-75cc6d9acd1c | None        | enroll             | False       |
| a4a53134-1690-45c6-93ee-b96c31dbfc54 | openshift-master-0 | a4a53134-1690-45c6-93ee-b96c31dbfc54 | None        | enroll             | False       |
| b2d87e8e-563e-4da4-96f4-99bcb754f7b1 | openshift-master-1 | b2d87e8e-563e-4da4-96f4-99bcb754f7b1 | None        | enroll             | False       |
+--------------------------------------+--------------------+--------------------------------------+-------------+--------------------+-------------+
~~~

> **NOTE**: You'll also notice that the IP address for Ironic has changed, it's now *172.22.0.3*, whereas when we were deploying it was *172.22.0.2*, only because it's now running on the cluster, no longer on the (long-since-deleted) bootstrap VM.

So what's the problem here? Well, remember when we had to add the static route for our bootstrap node, we now have to do this for our master nodes, where the Ironic pods run; essentially the Ironic pods on the masters can not talk to the out of band management (IPMI) interfaces for our nodes.

Let's fix thaat - for each master, force the static route:

~~~bash
kni@provisioner$ for i in {2..4}; do ssh -oStrictHostKeyChecking=no \
	core@192.168.111.$i sudo ip route add 192.0.2.0/24 via 192.168.111.11; done

Warning: Permanently added '192.168.111.2' (ECDSA) to the list of known hosts.
Warning: Permanently added '192.168.111.3' (ECDSA) to the list of known hosts.
Warning: Permanently added '192.168.111.4' (ECDSA) to the list of known hosts.
~~~

Eventually, Ironic will re-poll the hosts, and the baremetal-operator will re-sync with Ironic now that it's able to speak with IPMI, but the quickest way is to simply restart all of the metal3 pods:

~~~bash
kni@provisioner$ oc delete pod \
	$(oc get pods -n openshift-machine-api | awk '/metal3/ {print $1;}') \
	-n openshift-machine-api

pod "metal3-5d5c75dc4b-jmgk8" deleted
~~~

Now we can wait for the cluster operator to restart the pod again before proceeding, note that it may take a few minutes:

~~~bash
kni@provisioner$ oc wait --for condition=Ready pod \
	$(oc get pods -n openshift-machine-api | awk '/metal3/ {print $1;}') \
	--timeout=240s -n openshift-machine-api

pod/metal3-5d5c75dc4b-qrlsf condition met
~~~

<font color="red">
> **NOTE**: We have **only** had to do this due to the network routing issue in the lab environment, for customer production environments they will not have this issue.
</font>

<br />Now if we re-run our checks on the   hosts:

~~~bash
kni@provisioner$ $ oc get baremetalhosts -n openshift-machine-api
NAME                 STATUS   PROVISIONING STATUS      CONSUMER       BMC                  HARDWARE PROFILE   ONLINE   ERROR
openshift-master-0   OK       externally provisioned   kni-master-0   ipmi://192.0.2.221                      true
openshift-master-1   OK       externally provisioned   kni-master-1   ipmi://192.0.2.222                      true
openshift-master-2   OK       externally provisioned   kni-master-2   ipmi://192.0.2.223                      true

kni@provisioner$ openstack baremetal node list
+--------------------------------------+--------------------+--------------------------------------+-------------+--------------------+-------------+
| UUID                                 | Name               | Instance UUID                        | Power State | Provisioning State | Maintenance |
+--------------------------------------+--------------------+--------------------------------------+-------------+--------------------+-------------+
| 11b8ec0f-6531-4eea-aef6-64cdf746056a | openshift-master-0 | 11b8ec0f-6531-4eea-aef6-64cdf746056a | power on    | active             | False       |
| 62dceb1c-6a5b-4f92-8fb1-5c50bcd0cea7 | openshift-master-1 | 62dceb1c-6a5b-4f92-8fb1-5c50bcd0cea7 | power on    | active             | False       |
| 5c4b71d3-6b3a-4e19-aa62-5c521526bdc8 | openshift-master-2 | 5c4b71d3-6b3a-4e19-aa62-5c521526bdc8 | power on    | active             | False       |
+--------------------------------------+--------------------+--------------------------------------+-------------+--------------------+-------------+
~~~

Great! All looks good with the baremetal nodes! However, there's one more step for us to do...

Now that we have our baremetal hosts registered with the baremetal operator, we need to tell OpenShift which `Node` is which. Every computer within a Kubernetes environment is considered a `Node`, but with OpenShift 4.0+ the cluster is more aware of the underlying infrastructure, so it can make adjustments such as scaling the cluster, adding new nodes, and deleting them. OpenShift utilises the concept of `Machines` and `MachineSets` to help it understand the different types of underlying infrastructure, including public cloud platforms like AWS. A `Machine` is a fundamental unit that describes the host for a `Node`.

When we registered our baremetal hosts we created corresponding `Machine` objects (see **CONSUMER**) that are linked to our `BareMetalHost` objects:

~~~bash
kni@provisioner$ oc get baremetalhosts -n openshift-machine-api | awk \
	'{print $1 "  " $5;}' | column -t

NAME                CONSUMER
openshift-master-0  kni-master-0
openshift-master-1  kni-master-1
openshift-master-2  kni-master-2
~~~ 

However, all of the `Nodes`, i.e. the OpenShift/Kubernetes nodes that are our masters, are not currently linked to their corresponding `Machine`. You can verify this in the UI too - if you open up your OpenShift console again and scroll down to '**Compute**' on the left hand side and select '**Nodes**', you'll notice that each of the nodes doesn't have a corresponding `Machine` reference:

<img src="img/no-machine.png"/>

Furthermore, each `BareMetalHost` doesn't have a `Node` assigned, verified by selecting the '**Machines**' option in the same left hand side menu, note that you will need to select **all-projects** from the 'project' drop down at the top to see these entries:

<img src="img/no-node.png"/>


Thankfully we have a handy script that can link these for us:

~~~
kni@provisioner$ cd ~/dev-scripts/
kni@provisioner$ for i in {0..2}; \
	do ./link-machine-and-node.sh kni-master-$i master-$i ;done
(...)
~~~

> **NOTE**: There's a lot of output from the above command, but it's simply running a `patch` on each `Machine` object to link them together.

Now, if you ask OpenShift for the details of one of the `Machines` you can see how it's all connected together, noting the bits we've added for clarity:

~~~bash
kni@provisioner$$ oc get machine/kni-master-0 -n openshift-machine-api -o yaml
apiVersion: machine.openshift.io/v1beta1
kind: Machine
metadata:
  annotations:
    metal3.io/BareMetalHost: openshift-machine-api/openshift-master-0
  creationTimestamp: "2019-08-21T18:23:44Z"          ^^ this is the **BareMetalHost**
  finalizers:
  - machine.machine.openshift.io
  generation: 1
  labels:
    machine.openshift.io/cluster-api-cluster: kni
    machine.openshift.io/cluster-api-machine-role: master
    machine.openshift.io/cluster-api-machine-type: master
  name: kni-master-0                                     <--- this is the **machine**
  namespace: openshift-machine-api
  resourceVersion: "35928"
  selfLink: /apis/machine.openshift.io/v1beta1/namespaces/openshift-machine-api/machines/kni-master-0
  uid: cf7ded7d-c440-11e9-8e1b-525400427ae5
spec:
  metadata:
    creationTimestamp: null
  providerSpec:
    value:
      hostSelector: {}
      image:
        checksum: http://172.22.0.3:6180/images/rhcos-42.80.20190725.1-openstack.qcow2/rhcos-42.80.20190725.1-compressed.qcow2.md5sum
        url: http://172.22.0.3:6180/images/rhcos-42.80.20190725.1-openstack.qcow2/rhcos-42.80.20190725.1-compressed.qcow2
      metadata:
        creationTimestamp: null
      userData:
        name: master-user-data
status:
  addresses:
  - address: 192.168.111.2
    type: InternalIP
  - address: master-0
    type: Hostname
  - address: master-0
    type: InternalDNS
  lastUpdated: "2019-08-21T20:11:11Z"
  nodeRef:
    kind: Node
    name: master-0                                      <--- this is the **node**
    uid: 92761430-c440-11e9-8e1b-525400427ae5
~~~

You can also see this represented in the OpenShift console if you want to take a look. Open up your web browser again and navigate to '**Compute**' --> '**Machines**' (see the '**Node**' references), you'll need to make sure that you either select '**all-projects**' or '**openshift-machine-api**' in the Project drop down:

<img src="img/console-machines.png"/>

And then '**Compute**' --> '**Bare Metal Hosts**':

<img src="img/console-baremetal.png"/>
