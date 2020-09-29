#**Create OpenShift Cluster***



~~~bash
[cloud-user@provision scripts]$ cat install-config.yaml
apiVersion: v1
baseDomain: students.osp.opentlc.com
metadata:
  name: schmaustech
networking:
  networkType: OVNKubernetes
  machineCIDR: 10.20.0.0/24
compute:
- name: worker
  replicas: 2
controlPlane:
  name: master
  replicas: 3
  platform:
    baremetal: {}
platform:
  baremetal:
    provisioningNetworkCIDR: 172.22.0.0/24
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


~~~bash
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6200 -Uadmin -Predhat chassis power off
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6201 -Uadmin -Predhat chassis power off
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6202 -Uadmin -Predhat chassis power off
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6203 -Uadmin -Predhat chassis power off
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6204 -Uadmin -Predhat chassis power off
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6205 -Uadmin -Predhat chassis power off
~~~

~~~bash
  mkdir `pwd`/ocp
  cp `pwd`/install-config.yaml `pwd`/ocp
  `pwd`/openshift-baremetal-install --dir=ocp --log-level debug create manifest
  cp `pwd`/cluster-network-03-config.yml `pwd`/ocp/manifests/cluster-network-03-config.yml
  `pwd`/openshift-baremetal-install --dir=ocp --log-level debug create cluster
~~~




Confirm install-config.yml looks right
Set export variables
Run openshift-baremetal-install command to create cluster
Open another terminal and watch the log or view the boostrap VM
Show how to see what is happening in bootstrap vm
