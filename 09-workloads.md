# **Running Workloads in the Environment**

Let's get into actually testing some workloads on our environment... we've spent all of this time building it up but we haven't even proven that it's working properly yet! In this section we're going to be deploying some pods as well as deploying a VM using OpenShift virtualization.

## Deploying a Container

OK, so this is likely something that you've all done before, and it's hardly very exciting, but let's have a little bit of fun. Let's deploy a nifty little application inside of a pod and use it to verify that the OpenShift cluster is functioning properly; this will involve building an application from source and exposing it to your web-browser. We'll use the s2i (source to image) container type:

~~~bash
[lab-user@provision ~]$ oc project default
Already on project "default" on server "https://api.schmaustech.dynamic.opentlc.com:6443".
[lab-user@provision ~]$ oc new-app nodeshift/centos7-s2i-nodejs:12.x~https://github.com/vrutkovs/DuckHunt-JS
--> Found container image 5b0b75b (11 months old) from Docker Hub for "nodeshift/centos7-s2i-nodejs:12.x"

    Node.js 12.12.0 
    --------------- 
    Node.js  available as docker container is a base platform for building and running various Node.js  applications and frameworks. Node.js is a platform built on Chrome's JavaScript runtime for easily building fast, scalable network applications. Node.js uses an event-driven, non-blocking I/O model that makes it lightweight and efficient, perfect for data-intensive real-time applications that run across distributed devices.

    Tags: builder, nodejs, nodejs-12.12.0

    * An image stream tag will be created as "centos7-s2i-nodejs:12.x" that will track the source image
    * A source build using source code from https://github.com/vrutkovs/DuckHunt-JS will be created
      * The resulting image will be pushed to image stream tag "duckhunt-js:latest"
      * Every time "centos7-s2i-nodejs:12.x" changes a new build will be triggered

--> Creating resources ...
    buildconfig.build.openshift.io "duckhunt-js" created
    deployment.apps "duckhunt-js" created
    service "duckhunt-js" created
--> Success
    Build scheduled, use 'oc logs -f bc/duckhunt-js' to track its progress.
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/duckhunt-js' 
    Run 'oc status' to view your app.
~~~

Now our application will build from source, you can watch it happen with:

~~~bash
[lab-user@provision ~]$ oc logs duckhunt-js-1-build -f
Caching blobs under "/var/cache/blobs".
Getting image source signatures
Copying blob sha256:d8d02d45731499028db01b6fa35475f91d230628b4e25fab8e3c015594dc3261
Copying blob sha256:a11069b6e24573a516c9d542d7ed33adc115ebcde49101037d153958d3bc2e01
(...)
Writing manifest to image destination
Storing signatures
Successfully pushed image-registry.openshift-image-registry.svc:5000/default/duckhunt-js@sha256:a2eff4eca82019b2752bfbe7ab0f7dcb6875cc9a6ec16b09f351dd933612300b
Push successful
~~~

Now you can check if the Duckhunt pod has finished building and is running:

~~~bash
[lab-user@provision ~]$ oc get pods
NAME                          READY   STATUS      RESTARTS   AGE
duckhunt-js-1-build           0/1     Completed   0          2m21s
duckhunt-js-cb885554b-qqmw2   1/1     Running     0          39s
~~~

Now expose the application (via the service) so we can route to it from the outside...

~~~bash
[lab-user@provision ~]$ oc expose svc/duckhunt-js
route.route.openshift.io/duckhunt-js exposed

[lab-user@provision ~]$ oc get route duckhunt-js
NAME          HOST/PORT                                                       PATH   SERVICES      PORT       TERMINATION   WILDCARD
duckhunt-js   duckhunt-js-default.apps.schmaustech.dynamic.opentlc.com          duckhunt-js   8080-tcp                 None
~~~

Now you should be able to open up the application in the same browser that you're using for access to the OpenShift console. Copy and paste the host address from above and you should now be able to have a quick play with this... good luck ;-

<img src="img/duckhunt.png"/>

Now, if you can tear yourself away from the game, let's build a VM...

## Deploying a Virtual Machine (STOP - Work in Progress!)

If you recall back in the previous deploy OpenShift virtualization lab we went ahead and installed the OpenShift virtualization operater and created a virtualization cluster.  Further we went ahead and configured an external bridge so that are virtual machines can be connected to the outside network.  Therefore we are now at the point where we can launch a virtual machine.  To do this we will use the following yaml file below.  Lets go ahead and create the file:

~~~bash
[lab-user@provision scripts]$ cat << EOF > ~/virtualmachine-cirros.yaml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  annotations:
    kubevirt.io/latest-observed-api-version: v1alpha3
    kubevirt.io/storage-observed-api-version: v1alpha3
    name.os.template.kubevirt.io/silverblue32: Fedora 31 or higher
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachines/cirros
  resourceVersion: '95007'
  name: cirros
  uid: 6cef319d-b301-4004-9369-004d8408c509
  creationTimestamp: '2020-10-16T00:07:48Z'
  generation: 6
  managedFields:
    - apiVersion: kubevirt.io/v1alpha3
      fieldsType: FieldsV1
      fieldsV1:
        'f:metadata':
          'f:annotations':
            .: {}
            'f:name.os.template.kubevirt.io/silverblue32': {}
          'f:labels':
            'f:os.template.kubevirt.io/silverblue32': {}
            'f:vm.kubevirt.io/template.version': {}
            'f:vm.kubevirt.io/template.namespace': {}
            'f:app': {}
            .: {}
            'f:workload.template.kubevirt.io/desktop': {}
            'f:vm.kubevirt.io/template.revision': {}
            'f:flavor.template.kubevirt.io/small': {}
            'f:vm.kubevirt.io/template': {}
        'f:spec':
          .: {}
          'f:dataVolumeTemplates': {}
          'f:template':
            .: {}
            'f:metadata':
              .: {}
              'f:labels':
                .: {}
                'f:flavor.template.kubevirt.io/small': {}
                'f:kubevirt.io/domain': {}
                'f:kubevirt.io/size': {}
                'f:os.template.kubevirt.io/silverblue32': {}
                'f:vm.kubevirt.io/name': {}
                'f:workload.template.kubevirt.io/desktop': {}
            'f:spec':
              .: {}
              'f:domain':
                .: {}
                'f:cpu':
                  .: {}
                  'f:cores': {}
                  'f:sockets': {}
                  'f:threads': {}
                'f:devices':
                  .: {}
                  'f:autoattachPodInterface': {}
                  'f:disks': {}
                  'f:inputs': {}
                  'f:interfaces': {}
                  'f:networkInterfaceMultiqueue': {}
                  'f:rng': {}
                'f:machine':
                  .: {}
                  'f:type': {}
                'f:resources':
                  .: {}
                  'f:requests':
                    .: {}
                    'f:memory': {}
              'f:evictionStrategy': {}
              'f:hostname': {}
              'f:networks': {}
              'f:terminationGracePeriodSeconds': {}
              'f:volumes': {}
      manager: Mozilla
      operation: Update
      time: '2020-10-16T00:07:48Z'
    - apiVersion: kubevirt.io/v1alpha3
      fieldsType: FieldsV1
      fieldsV1:
        'f:metadata':
          'f:annotations':
            'f:kubevirt.io/latest-observed-api-version': {}
            'f:kubevirt.io/storage-observed-api-version': {}
        'f:status':
          .: {}
          'f:conditions': {}
          'f:created': {}
          'f:ready': {}
      manager: virt-controller
      operation: Update
      time: '2020-10-16T00:10:30Z'
    - apiVersion: kubevirt.io/v1alpha3
      fieldsType: FieldsV1
      fieldsV1:
        'f:spec':
          'f:running': {}
      manager: virt-api
      operation: Update
      time: '2020-10-16T00:13:04Z'
  namespace: default
  labels:
    app: cirros
    flavor.template.kubevirt.io/small: 'true'
    os.template.kubevirt.io/silverblue32: 'true'
    vm.kubevirt.io/template: fedora-desktop-small-v0.11.3
    vm.kubevirt.io/template.namespace: openshift
    vm.kubevirt.io/template.revision: '1'
    vm.kubevirt.io/template.version: v0.11.3
    workload.template.kubevirt.io/desktop: 'true'
spec:
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1alpha1
      kind: DataVolume
      metadata:
        creationTimestamp: null
        name: cirros-rootdisk
      spec:
        pvc:
          accessModes:
            - ReadWriteMany
          resources:
            requests:
              storage: 5Gi
          storageClassName: ocs-storagecluster-ceph-rbd
          volumeMode: Block
        source:
          http:
            url: >-
              http://download.cirros-cloud.net/0.5.1/cirros-0.5.1-x86_64-disk.img
      status: {}
  running: false
  template:
    metadata:
      creationTimestamp: null
      labels:
        flavor.template.kubevirt.io/small: 'true'
        kubevirt.io/domain: cirros
        kubevirt.io/size: small
        os.template.kubevirt.io/silverblue32: 'true'
        vm.kubevirt.io/name: cirros
        workload.template.kubevirt.io/desktop: 'true'
    spec:
      domain:
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          autoattachPodInterface: false
          disks:
            - bootOrder: 1
              disk:
                bus: virtio
              name: rootdisk
            - disk:
                bus: virtio
              name: cloudinitdisk
          inputs:
            - bus: virtio
              name: tablet
              type: tablet
          interfaces:
            - bridge: {}
              model: virtio
              name: nic-0
          networkInterfaceMultiqueue: true
          rng: {}
        machine:
          type: pc-q35-rhel8.2.0
        resources:
          requests:
            memory: 2Gi
      evictionStrategy: LiveMigrate
      hostname: cirrostest
      networks:
        - multus:
            networkName: brext
          name: nic-0
      terminationGracePeriodSeconds: 180
      volumes:
        - dataVolume:
            name: cirros-rootdisk
          name: rootdisk
        - cloudInitNoCloud:
            userData: |
              #cloud-config
              name: default
              hostname: cirrostest
              runcmd:
                - ip a add 10.20.0.50/24 dev eth0
                - ip r add 0.0.0.0/0 10.20.0.1 dev eth0
          name: cloudinitdisk
status:
  conditions:
    - lastProbeTime: null
      lastTransitionTime: '2020-10-16T00:09:26Z'
      status: 'True'
      type: Ready
  created: true
  ready: true
EOF
~~~

Lets take a look at the file we just created:

~~~bash 
[lab-user@provision scripts]$ cat ~/virtualmachine-cirros.yaml 
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  annotations:
    kubevirt.io/latest-observed-api-version: v1alpha3
    kubevirt.io/storage-observed-api-version: v1alpha3
    name.os.template.kubevirt.io/silverblue32: Fedora 31 or higher
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachines/cirros
  resourceVersion: '95007'
  name: cirros
  uid: 6cef319d-b301-4004-9369-004d8408c509
  creationTimestamp: '2020-10-16T00:07:48Z'
  generation: 6
  managedFields:
    - apiVersion: kubevirt.io/v1alpha3
      fieldsType: FieldsV1
      fieldsV1:
        'f:metadata':
          'f:annotations':
            .: {}
            'f:name.os.template.kubevirt.io/silverblue32': {}
          'f:labels':
            'f:os.template.kubevirt.io/silverblue32': {}
            'f:vm.kubevirt.io/template.version': {}
            'f:vm.kubevirt.io/template.namespace': {}
            'f:app': {}
            .: {}
            'f:workload.template.kubevirt.io/desktop': {}
            'f:vm.kubevirt.io/template.revision': {}
            'f:flavor.template.kubevirt.io/small': {}
            'f:vm.kubevirt.io/template': {}
        'f:spec':
          .: {}
          'f:dataVolumeTemplates': {}
          'f:template':
            .: {}
            'f:metadata':
              .: {}
              'f:labels':
                .: {}
                'f:flavor.template.kubevirt.io/small': {}
                'f:kubevirt.io/domain': {}
                'f:kubevirt.io/size': {}
                'f:os.template.kubevirt.io/silverblue32': {}
                'f:vm.kubevirt.io/name': {}
                'f:workload.template.kubevirt.io/desktop': {}
            'f:spec':
              .: {}
              'f:domain':
                .: {}
                'f:cpu':
                  .: {}
                  'f:cores': {}
                  'f:sockets': {}
                  'f:threads': {}
                'f:devices':
                  .: {}
                  'f:autoattachPodInterface': {}
                  'f:disks': {}
                  'f:inputs': {}
                  'f:interfaces': {}
                  'f:networkInterfaceMultiqueue': {}
                  'f:rng': {}
                'f:machine':
                  .: {}
                  'f:type': {}
                'f:resources':
                  .: {}
                  'f:requests':
                    .: {}
                    'f:memory': {}
              'f:evictionStrategy': {}
              'f:hostname': {}
              'f:networks': {}
              'f:terminationGracePeriodSeconds': {}
              'f:volumes': {}
      manager: Mozilla
      operation: Update
      time: '2020-10-16T00:07:48Z'
    - apiVersion: kubevirt.io/v1alpha3
      fieldsType: FieldsV1
      fieldsV1:
        'f:metadata':
          'f:annotations':
            'f:kubevirt.io/latest-observed-api-version': {}
            'f:kubevirt.io/storage-observed-api-version': {}
        'f:status':
          .: {}
          'f:conditions': {}
          'f:created': {}
          'f:ready': {}
      manager: virt-controller
      operation: Update
      time: '2020-10-16T00:10:30Z'
    - apiVersion: kubevirt.io/v1alpha3
      fieldsType: FieldsV1
      fieldsV1:
        'f:spec':
          'f:running': {}
      manager: virt-api
      operation: Update
      time: '2020-10-16T00:13:04Z'
  namespace: default
  labels:
    app: cirros
    flavor.template.kubevirt.io/small: 'true'
    os.template.kubevirt.io/silverblue32: 'true'
    vm.kubevirt.io/template: fedora-desktop-small-v0.11.3
    vm.kubevirt.io/template.namespace: openshift
    vm.kubevirt.io/template.revision: '1'
    vm.kubevirt.io/template.version: v0.11.3
    workload.template.kubevirt.io/desktop: 'true'
spec:
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1alpha1
      kind: DataVolume
      metadata:
        creationTimestamp: null
        name: cirros-rootdisk
      spec:
        pvc:
          accessModes:
            - ReadWriteMany
          resources:
            requests:
              storage: 5Gi
          storageClassName: ocs-storagecluster-ceph-rbd
          volumeMode: Block
        source:
          http:
            url: >-
              http://download.cirros-cloud.net/0.5.1/cirros-0.5.1-x86_64-disk.img
      status: {}
  running: false
  template:
    metadata:
      creationTimestamp: null
      labels:
        flavor.template.kubevirt.io/small: 'true'
        kubevirt.io/domain: cirros
        kubevirt.io/size: small
        os.template.kubevirt.io/silverblue32: 'true'
        vm.kubevirt.io/name: cirros
        workload.template.kubevirt.io/desktop: 'true'
    spec:
      domain:
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          autoattachPodInterface: false
          disks:
            - bootOrder: 1
              disk:
                bus: virtio
              name: rootdisk
            - disk:
                bus: virtio
              name: cloudinitdisk
          inputs:
            - bus: virtio
              name: tablet
              type: tablet
          interfaces:
            - bridge: {}
              model: virtio
              name: nic-0
          networkInterfaceMultiqueue: true
          rng: {}
        machine:
          type: pc-q35-rhel8.2.0
        resources:
          requests:
            memory: 2Gi
      evictionStrategy: LiveMigrate
      hostname: cirrostest
      networks:
        - multus:
            networkName: brext
          name: nic-0
      terminationGracePeriodSeconds: 180
      volumes:
        - dataVolume:
            name: cirros-rootdisk
          name: rootdisk
        - cloudInitNoCloud:
            userData: |
              #cloud-config
              name: default
              hostname: cirrostest
              runcmd:
                - ip a add 10.20.0.50/24 dev eth0
                - ip r add 0.0.0.0/0 10.20.0.1 dev eth0
          name: cloudinitdisk
status:
  conditions:
    - lastProbeTime: null
      lastTransitionTime: '2020-10-16T00:09:26Z'
      status: 'True'
      type: Ready
  created: true
  ready: true
~~~

Now lets create the virtual machine:

~~~bash
[lab-user@provision scripts]$ oc create -f ~/virtualmachine-cirros.yaml 
virtualmachine.kubevirt.io/cirros created
~~~

If we run a oc get pods for the default namespace we will see an importer container.  Once it is running we can watch the logs:

~~~bash
NAME                       READY   STATUS              RESTARTS   AGE
importer-cirros-rootdisk   0/1     ContainerCreating   0          9s
[lab-user@provision scripts]$ oc get pods
NAME                       READY   STATUS    RESTARTS   AGE
importer-cirros-rootdisk   1/1     Running   0          15s
[lab-user@provision scripts]$ oc logs importer-cirros-rootdisk -f
I1016 00:26:19.628409       1 importer.go:51] Starting importer
I1016 00:26:19.629363       1 importer.go:112] begin import process
E1016 00:26:20.098193       1 http-datasource.go:329] http: expected status code 200, got 403
I1016 00:26:20.245269       1 data-processor.go:277] Calculating available size
I1016 00:26:20.246168       1 data-processor.go:285] Checking out block volume size.
I1016 00:26:20.246182       1 data-processor.go:297] Request image size not empty.
I1016 00:26:20.246190       1 data-processor.go:302] Target size 5Gi.
I1016 00:26:20.246277       1 util.go:37] deleting file: /scratch/lost+found
I1016 00:26:20.276044       1 data-processor.go:206] New phase: TransferScratch
I1016 00:26:20.276180       1 util.go:161] Writing data...
I1016 00:26:20.671715       1 data-processor.go:206] New phase: Process
I1016 00:26:20.671740       1 data-processor.go:206] New phase: Convert
I1016 00:26:20.671745       1 data-processor.go:212] Validating image
I1016 00:26:22.072371       1 data-processor.go:206] New phase: Resize
I1016 00:26:22.073309       1 data-processor.go:206] New phase: Complete
I1016 00:26:22.073367       1 util.go:37] deleting file: /scratch/tmpimage
I1016 00:26:22.075290       1 importer.go:175] Import complete
~~~

The virtual machine has been created but not started.  Lets install virtctl before we proceed:

~~~bash
[lab-user@provision scripts]$ sudo dnf -y install kubevirt-virtctl
Updating Subscription Management repositories.
Red Hat Enterprise Linux 8 for x86_64 - BaseOS (RPMs)                                                                                                                              6.7 kB/s | 2.4 kB     00:00    
Red Hat Enterprise Linux 8 for x86_64 - AppStream (RPMs)                                                                                                                           7.7 kB/s | 2.8 kB     00:00    
Red Hat OpenStack Platform 15 for RHEL 8 x86_64 (RPMs)                                                                                                                             6.8 kB/s | 2.4 kB     00:00    
Red Hat OpenStack Platform 15 Tools for RHEL 8 x86_64 (RPMs)                                                                                                                       6.1 kB/s | 2.1 kB     00:00    
Red Hat Ansible Engine 2 for RHEL 8 x86_64 (RPMs)                                                                                                                                  6.6 kB/s | 2.3 kB     00:00    
Red Hat Container Native Virtualization 2.3 for RHEL 8 x86_64 (RPMs)                                                                                                               6.7 kB/s | 2.3 kB     00:00    
Dependencies resolved.
===================================================================================================================================================================================================================
 Package                                           Architecture                            Version                                           Repository                                                       Size
===================================================================================================================================================================================================================
Installing:
 kubevirt-virtctl                                  x86_64                                  0.26.1-15.el8                                     cnv-2.3-for-rhel-8-x86_64-rpms                                  8.1 M

Transaction Summary
===================================================================================================================================================================================================================
Install  1 Package

Total download size: 8.1 M
Installed size: 40 M
Downloading Packages:
kubevirt-virtctl-0.26.1-15.el8.x86_64.rpm                                                                                                                                          9.9 MB/s | 8.1 MB     00:00    
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                                              9.9 MB/s | 8.1 MB     00:00     
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                                           1/1 
  Installing       : kubevirt-virtctl-0.26.1-15.el8.x86_64                                                                                                                                                     1/1 
  Verifying        : kubevirt-virtctl-0.26.1-15.el8.x86_64                                                                                                                                                     1/1 
Installed products updated.

Installed:
  kubevirt-virtctl-0.26.1-15.el8.x86_64                                                                                                                                                                            

Complete!
~~~

Now we will use virtctl to start the virtual machine:

~~~bash
[lab-user@provision scripts]$ virtctl start cirros
VM cirros was scheduled to start
~~~

Give the virtualmachine a few minutes to start up.  You can test if it is up by trying to hit the IP address of the host:

~~~bash
[lab-user@provision scripts]$ ping 10.20.0.50
PING 10.20.0.50 (10.20.0.50) 56(84) bytes of data.
64 bytes from 10.20.0.50: icmp_seq=1 ttl=64 time=1.43 ms
64 bytes from 10.20.0.50: icmp_seq=2 ttl=64 time=0.797 ms
64 bytes from 10.20.0.50: icmp_seq=3 ttl=64 time=0.195 ms
64 bytes from 10.20.0.50: icmp_seq=4 ttl=64 time=0.179 ms
^C
--- 10.20.0.50 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 58ms
rtt min/avg/max/mdev = 0.179/0.649/1.428/0.514 ms
~~~



**Success**, we're done! Congratulations... if you've made it this far you've deployed KNI from the ground up, deployed Ceph via Rook, Container Native Virtualisation (CNV), and tested the solution with pods and VM's via the CLI and the OpenShift dashboard! I'd like to ***thank you*** for attending this lab; I hope that it was a valuable use of your time and that your learnt a lot from doing it. Please do let us know if there's anything else we can do to support you! There's also a CNV-based lab here at Red Hat Tech Exchange if you're keen on exploring it further.
