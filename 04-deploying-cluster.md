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
  `pwd`/openshift-baremetal-install --dir=ocp --log-level debug create cluster
~~~


~~~bash

~~~

