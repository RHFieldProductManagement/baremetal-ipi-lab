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

Before running deployment power off nodes

~~~bash
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6200 -Uadmin -Predhat chassis power off
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6201 -Uadmin -Predhat chassis power off
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6202 -Uadmin -Predhat chassis power off
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6203 -Uadmin -Predhat chassis power off
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6204 -Uadmin -Predhat chassis power off
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6205 -Uadmin -Predhat chassis power off
~~~

Run the deployment using the install-config.yaml

~~~bash
  mkdir `pwd`/ocp
  cp `pwd`/install-config.yaml `pwd`/ocp
  `pwd`/openshift-baremetal-install --dir=ocp --log-level debug create manifest
  cp `pwd`/cluster-network-03-config.yml `pwd`/ocp/manifests/cluster-network-03-config.yml
  `pwd`/openshift-baremetal-install --dir=ocp --log-level debug create cluster
~~~

Show deployment log

