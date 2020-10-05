#**Create OpenShift Cluster***

In the previous lab we configured a disconnected registry and httpd cache for RHCOS images.  This will allow us to test our disconnected OpenShift deploy in this section of the lab.  But before we begin lets look at a few parts of the install-config.yaml configuration file.  If we can out the file we can see there are various sections and attributes:

~~~bash
[cloud-user@provision scripts]$ cat install-config.yaml
apiVersion: v1
baseDomain: students.osp.opentlc.com
metadata:
  name: schmaustech <===CLUSTER NAME
networking:
  networkType: OVNKubernetes <===NETWORK SDN TO USE ON DEPLOY
  machineCIDR: 10.20.0.0/24 <=== EXTERNAL/BAREMETAL NETWORK
compute:
- name: worker
  replicas: 2 <===NUMBER OF WORKERS ON DEPLOYMENT
controlPlane:
  name: master
  replicas: 3 <===NUMBER OF MASTERS ON DEPLOYMENT
  platform:
    baremetal: {}
platform:
  baremetal:
    provisioningNetworkCIDR: 172.22.0.0/24 <=== NETWORK OF PROVISIONING NETWORK
    provisioningNetworkInterface: ens3
    apiVIP: 10.20.0.110
    ingressVIP: 10.20.0.112
    dnsVIP: 10.20.0.111
    bootstrapOSImage: http://10.20.0.2/images/rhcos-45.82.202008010929-0-qemu.x86_64.qcow2.gz?sha256=c9e2698d0f3bcc48b7c66d7db901266abf27ebd7474b6719992de2d8db96995a
    clusterOSImage: http://10.20.0.2/images/rhcos-45.82.202008010929-0-openstack.x86_64.qcow2.gz?sha256=359e7c3560fdd91e64cd0d8df6a172722b10e777aef38673af6246f14838ab1a
    hosts:
      - name: master-0
        role: master
        bmc:
          address: ipmi://10.20.0.3:6204
          username: admin
          password: redhat
        bootMACAddress: de:ad:be:ef:00:40
        hardwareProfile: openstack
      - name: master-1
        role: master
        bmc:
          address: ipmi://10.20.0.3:6201
          username: admin
          password: redhat
        bootMACAddress: de:ad:be:ef:00:41
        hardwareProfile: openstack
      - name: master-2
        role: master
        bmc:
          address: ipmi://10.20.0.3:6200
          username: admin
          password: redhat
        bootMACAddress: de:ad:be:ef:00:42
        hardwareProfile: openstack
      - name: worker-0
        role: worker
        bmc:
          address: ipmi://10.20.0.3:6205
          username: admin
          password: redhat
        bootMACAddress: de:ad:be:ef:00:50
        hardwareProfile: openstack
      - name: worker-1
        role: worker
        bmc:
          address: ipmi://10.20.0.3:6202
          username: admin
          password: redhat
        bootMACAddress: de:ad:be:ef:00:51
        hardwareProfile: openstack
sshKey: 'ssh-rsa REDACTED SSH KEY cloud-user@provision'
imageContentSources:
- mirrors:
  - provision.schmaustech.students.osp.opentlc.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - provision.schmaustech.students.osp.opentlc.com:5000/ocp4/openshift4
  source: registry.svc.ci.openshift.org/ocp/release
pullSecret: 'REDACTED PULL SECRET'
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
 REDACTED CERTIFICATE
  -----END CERTIFICATE-----
~~~

Now that we have examined the install-config.yaml we are ready to proceed with the deployment.  However before starting the deploy we should always make sure that the master and worker baremetal nodes we are going to use are in a powered off state:

~~~bash
[cloud-user@provision scripts]$ for i in 0 1 2 3 4 5
> do
> /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p620$i -Uadmin -Predhat chassis power off
> done
Chassis Power Control: Down/Off
Chassis Power Control: Down/Off
Chassis Power Control: Down/Off
Chassis Power Control: Down/Off
Chassis Power Control: Down/Off
Chassis Power Control: Down/Off
[cloud-user@provision scripts]$
~~~

Run the deployment using the install-config.yaml

~~~bash
[cloud-user@provision scripts]$ mkdir $HOME/scripts/ocp
[cloud-user@provision scripts]$ cp $HOME/scripts/install-config.yaml $HOME/scripts/ocp
[cloud-user@provision scripts]$ $HOME/scripts/openshift-baremetal-install --dir=ocp --log-level debug create manifests
DEBUG OpenShift Installer 4.5.9                    
DEBUG Built from commit 0d5c871ce7d03f3d03ab4371dc39916a5415cf5c 
DEBUG Fetching Master Machines...                  
DEBUG Loading Master Machines...                   
(...)
DEBUG   Loading Private Cluster Outbound Service... 
DEBUG   Loading Baremetal Config CR...             
DEBUG   Loading Image...                           
WARNING Discarding the Openshift Manifests that was provided in the target directory because its dependencies are dirty and it needs to be regenerated 
DEBUG   Fetching Install Config...                 
DEBUG   Reusing previously-fetched Install Config  
DEBUG   Fetching Cluster ID...                     
DEBUG   Reusing previously-fetched Cluster ID      
DEBUG   Fetching Kubeadmin Password...             
DEBUG   Generating Kubeadmin Password...           
DEBUG   Fetching OpenShift Install (Manifests)...  
DEBUG   Generating OpenShift Install (Manifests)... 
DEBUG   Fetching CloudCredsSecret...               
DEBUG   Generating CloudCredsSecret...             
DEBUG   Fetching KubeadminPasswordSecret...        
DEBUG   Generating KubeadminPasswordSecret...      
DEBUG   Fetching RoleCloudCredsSecretReader...     
DEBUG   Generating RoleCloudCredsSecretReader...   
DEBUG   Fetching Private Cluster Outbound Service... 
DEBUG   Generating Private Cluster Outbound Service... 
DEBUG   Fetching Baremetal Config CR...            
DEBUG   Generating Baremetal Config CR...          
DEBUG   Fetching Image...                          
DEBUG   Reusing previously-fetched Image           
DEBUG Generating Openshift Manifests...  
~~~

Note that generating the manifests would be done automatically if we just ran create cluster out of the gate.  However if you had additional configuration yamls this would be how you could add them in now.  If we look in the manifests directory we can see there are all sorts of configuration items.  Further yamls could be placed here for customizations of the cluster before actually kicking off the deploy:

~~~bash
[cloud-user@provision scripts]$ ls -l $HOME/scripts/ocp/manifests/
total 124
-rw-r-----. 1 cloud-user cloud-user  169 Oct  5 13:49 04-openshift-machine-config-operator.yaml
-rw-r-----. 1 cloud-user cloud-user 6383 Oct  5 13:49 cluster-config.yaml
-rw-r-----. 1 cloud-user cloud-user  165 Oct  5 13:49 cluster-dns-02-config.yml
-rw-r-----. 1 cloud-user cloud-user  581 Oct  5 13:49 cluster-infrastructure-02-config.yml
-rw-r-----. 1 cloud-user cloud-user  170 Oct  5 13:49 cluster-ingress-02-config.yml
-rw-r-----. 1 cloud-user cloud-user  513 Oct  5 13:49 cluster-network-01-crd.yml
-rw-r-----. 1 cloud-user cloud-user  272 Oct  5 13:49 cluster-network-02-config.yml
-rw-r-----. 1 cloud-user cloud-user  142 Oct  5 13:49 cluster-proxy-01-config.yaml
-rw-r-----. 1 cloud-user cloud-user  171 Oct  5 13:49 cluster-scheduler-02-config.yml
-rw-r-----. 1 cloud-user cloud-user  264 Oct  5 13:49 cvo-overrides.yaml
-rw-r-----. 1 cloud-user cloud-user 1335 Oct  5 13:49 etcd-ca-bundle-configmap.yaml
-rw-r-----. 1 cloud-user cloud-user 3958 Oct  5 13:49 etcd-client-secret.yaml
-rw-r-----. 1 cloud-user cloud-user  434 Oct  5 13:49 etcd-host-service-endpoints.yaml
-rw-r-----. 1 cloud-user cloud-user  271 Oct  5 13:49 etcd-host-service.yaml
-rw-r-----. 1 cloud-user cloud-user 4009 Oct  5 13:49 etcd-metric-client-secret.yaml
-rw-r-----. 1 cloud-user cloud-user 1359 Oct  5 13:49 etcd-metric-serving-ca-configmap.yaml
-rw-r-----. 1 cloud-user cloud-user 3917 Oct  5 13:49 etcd-metric-signer-secret.yaml
-rw-r-----. 1 cloud-user cloud-user  156 Oct  5 13:49 etcd-namespace.yaml
-rw-r-----. 1 cloud-user cloud-user  334 Oct  5 13:49 etcd-service.yaml
-rw-r-----. 1 cloud-user cloud-user 1336 Oct  5 13:49 etcd-serving-ca-configmap.yaml
-rw-r-----. 1 cloud-user cloud-user 3890 Oct  5 13:49 etcd-signer-secret.yaml
-rw-r-----. 1 cloud-user cloud-user  312 Oct  5 13:49 image-content-source-policy-0.yaml
-rw-r-----. 1 cloud-user cloud-user  307 Oct  5 13:49 image-content-source-policy-1.yaml
-rw-r-----. 1 cloud-user cloud-user  118 Oct  5 13:49 kube-cloud-config.yaml
-rw-r-----. 1 cloud-user cloud-user 1304 Oct  5 13:49 kube-system-configmap-root-ca.yaml
-rw-r-----. 1 cloud-user cloud-user 4134 Oct  5 13:49 machine-config-server-tls-secret.yaml
-rw-r-----. 1 cloud-user cloud-user 5705 Oct  5 13:49 openshift-config-secret-pul
~~~
 
 Finally we have now arrived at the point where we can run the create cluster command to deploy our baremetal cluster.   This process will take about ~60-70 minutes to complete:
  
~~~bash
[cloud-user@provision scripts]$ $HOME/scripts/openshift-baremetal-install --dir=ocp --log-level debug create cluster
DEBUG OpenShift Installer 4.5.9                    
DEBUG Built from commit 0d5c871ce7d03f3d03ab4371dc39916a5415cf5c 
DEBUG Fetching Metadata...                         
DEBUG Loading Metadata...                          
DEBUG   Loading Cluster ID...                      
DEBUG     Loading Install Config...                
DEBUG       Loading SSH Key...                     
DEBUG       Loading Base Domain...                 
DEBUG         Loading Platform...                  
DEBUG       Loading Cluster Name...                
DEBUG         Loading Base Domain...               
DEBUG         Loading Platform...                  
DEBUG       Loading Pull Secret...                 
DEBUG       Loading Platform...                    
DEBUG     Using Install Config loaded from state file 
DEBUG   Using Cluster ID loaded from state file
(...)
INFO Obtaining RHCOS image file from 'http://10.20.0.2/images/rhcos-45.82.202008010929-0-qemu.x86_64.qcow2.gz?sha256=c9e2698d0f3bcc48b7c66d7db901266abf27ebd7474b6719992de2d8db96995a' 
INFO The file was found in cache: /home/cloud-user/.cache/openshift-installer/image_cache/ad57fdbef98553f778ac17b95b094a1a. Reusing... 
INFO Consuming OpenShift Install (Manifests) from target directory 
(...)
DEBUG module.masters.ironic_node_v1.openshift-master-host[1]: Still creating... [2m10s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[0]: Still creating... [2m10s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[2]: Still creating... [2m10s elapsed] 
DEBUG module.bootstrap.libvirt_volume.bootstrap: Still creating... [2m10s elapsed] 
DEBUG module.bootstrap.libvirt_ignition.bootstrap: Still creating... [2m10s elapsed] 
DEBUG module.bootstrap.libvirt_volume.bootstrap: Creation complete after 2m17s [id=/var/lib/libvirt/images/schmaustech-mhnfj-bootstrap] 
DEBUG module.bootstrap.libvirt_ignition.bootstrap: Creation complete after 2m17s [id=/var/lib/libvirt/images/schmaustech-mhnfj-bootstrap.ign;5f7b64ba-cc8f-cec4-bf8c-47a4883c9bf6] 
DEBUG module.bootstrap.libvirt_domain.bootstrap: Creating... 
DEBUG module.bootstrap.libvirt_domain.bootstrap: Creation complete after 1s [id=008b263f-363d-4685-a2ca-e8852e3b5d05] 
(...)
DEBUG module.masters.ironic_node_v1.openshift-master-host[0]: Creation complete after 24m20s [id=63c12136-0605-4b0b-a2b3-b53b992b8189] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[1]: Still creating... [24m21s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[2]: Still creating... [24m21s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[1]: Still creating... [24m31s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[2]: Still creating... [24m31s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[2]: Still creating... [24m41s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[1]: Still creating... [24m41s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[2]: Creation complete after 24m41s [id=a84a8327-3ecc-440c-91a2-fcf6546ab1f1] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[1]: Still creating... [24m51s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[1]: Still creating... [25m1s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[1]: Creation complete after 25m2s [id=cb208530-0ff9-4946-bd11-b514190a56c1] 
DEBUG module.masters.data.ironic_introspection.openshift-master-introspection[0]: Refreshing state... 
DEBUG module.masters.data.ironic_introspection.openshift-master-introspection[2]: Refreshing state... 
DEBUG module.masters.data.ironic_introspection.openshift-master-introspection[1]: Refreshing state... 
DEBUG module.masters.ironic_allocation_v1.openshift-master-allocation[0]: Creating... 
DEBUG module.masters.ironic_allocation_v1.openshift-master-allocation[2]: Creating... 
DEBUG module.masters.ironic_allocation_v1.openshift-master-allocation[1]: Creating... 
DEBUG module.masters.ironic_allocation_v1.openshift-master-allocation[1]: Creation complete after 2s [id=5920c1cc-4e14-4563-b9a4-12618ca315ba] 
DEBUG module.masters.ironic_allocation_v1.openshift-master-allocation[2]: Creation complete after 3s [id=9f371836-b9bd-4bd1-9e6d-d604c6c9d1b8] 
DEBUG module.masters.ironic_allocation_v1.openshift-master-allocation[0]: Creation complete after 3s [id=0537b2e4-8ba4-42a4-9f93-0a138444ae42] 
DEBUG module.masters.ironic_deployment.openshift-master-deployment[0]: Creating... 
DEBUG module.masters.ironic_deployment.openshift-master-deployment[1]: Creating... 
DEBUG module.masters.ironic_deployment.openshift-master-deployment[2]: Creating... 
DEBUG module.masters.ironic_deployment.openshift-master-deployment[0]: Still creating... [10s elapsed] 
DEBUG module.masters.ironic_deployment.openshift-master-deployment[2]: Still creating... [10s elapsed] 
DEBUG module.masters.ironic_deployment.openshift-master-deployment[1]: Still creating... [10s elapsed] 
(...)
DEBUG module.masters.ironic_deployment.openshift-master-deployment[0]: Still creating... [9m0s elapsed] 
DEBUG module.masters.ironic_deployment.openshift-master-deployment[0]: Creation complete after 9m4s [id=63c12136-0605-4b0b-a2b3-b53b992b8189] 
DEBUG                                              
DEBUG Apply complete! Resources: 12 added, 0 changed, 0 destroyed. 
DEBUG OpenShift Installer 4.5.9                    
DEBUG Built from commit 0d5c871ce7d03f3d03ab4371dc39916a5415cf5c 
INFO Waiting up to 20m0s for the Kubernetes API at https://api.schmaustech.students.osp.opentlc.com:6443... 
INFO API v1.18.3+6c42de8 up                       
INFO Waiting up to 40m0s for bootstrapping to complete...
(...)
DEBUG Bootstrap status: complete                   
INFO Destroying the bootstrap resources... 
(...)
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.9 
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.9: downloading update 
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.9: 0% complete 
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.9: 41% complete 
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.9: 57% complete 
(...)
DEBUG Still waiting for the cluster to initialize: Some cluster operators are still updating: authentication, console, csi-snapshot-controller, ingress, kube-storage-version-migrator, monitoring 
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.9: 86% complete 
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.9: 86% complete 
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.9: 86% complete 
(...)
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.9: 92% complete 
DEBUG Cluster is initialized                       
INFO Waiting up to 10m0s for the openshift-console route to be created... 
DEBUG Route found in openshift-console namespace: console 
DEBUG Route found in openshift-console namespace: downloads 
DEBUG OpenShift console route is created           
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/cloud-user/scripts/ocp/auth/kubeconfig' 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.schmaustech.students.osp.opentlc.com 
INFO Login to the console with user: "kubeadmin", and password: "5VGM2-uMov3-4N2Vi-n5i3H" 
DEBUG Time elapsed per stage:                      
DEBUG     Infrastructure: 34m17s                   
DEBUG Bootstrap Complete: 35m56s                   
DEBUG  Bootstrap Destroy: 10s                      
DEBUG  Cluster Operators: 38m6s                    
INFO Time elapsed: 1h48m36s  
~~~


~~~bash
[cloud-user@provision ~]$ oc get nodes
NAME       STATUS   ROLES    AGE   VERSION
master-0   Ready    master   96m   v1.18.3+6c42de8
master-1   Ready    master   84m   v1.18.3+6c42de8
master-2   Ready    master   85m   v1.18.3+6c42de8
worker-0   Ready    worker   57m   v1.18.3+6c42de8
worker-1   Ready    worker   55m   v1.18.3+6c42de8
~~~

~~~bash
[cloud-user@provision ~]$ oc get clusteroperators
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.5.9     True        False         False      45m
cloud-credential                           4.5.9     True        False         False      131m
cluster-autoscaler                         4.5.9     True        False         False      64m
config-operator                            4.5.9     True        False         False      64m
console                                    4.5.9     True        False         False      48m
csi-snapshot-controller                    4.5.9     True        False         False      54m
dns                                        4.5.9     True        False         False      93m
etcd                                       4.5.9     True        False         False      83m
image-registry                             4.5.9     True        False         False      76m
ingress                                    4.5.9     True        False         False      54m
insights                                   4.5.9     True        False         False      79m
kube-apiserver                             4.5.9     True        False         False      82m
kube-controller-manager                    4.5.9     True        False         False      93m
kube-scheduler                             4.5.9     True        False         False      93m
kube-storage-version-migrator              4.5.9     True        False         False      55m
machine-api                                4.5.9     True        False         False      71m
machine-approver                           4.5.9     True        False         False      81m
machine-config                             4.5.9     True        False         False      42m
marketplace                                4.5.9     True        False         False      63m
monitoring                                 4.5.9     True        False         False      42m
network                                    4.5.9     True        False         False      96m
node-tuning                                4.5.9     True        False         False      95m
openshift-apiserver                        4.5.9     True        False         False      64m
openshift-controller-manager               4.5.9     True        False         False      78m
openshift-samples                          4.5.9     True        False         False      63m
operator-lifecycle-manager                 4.5.9     True        False         False      94m
operator-lifecycle-manager-catalog         4.5.9     True        False         False      94m
operator-lifecycle-manager-packageserver   4.5.9     True        False         False      64m
service-ca                                 4.5.9     True        False         False      95m
storage                                    4.5.9     True        False         False      78m
~~~



