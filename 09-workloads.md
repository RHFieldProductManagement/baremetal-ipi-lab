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

## Deploying a Virtual Machine

If you recall back in the previous deploy OpenShift virtualization lab we went ahead and installed the OpenShift virtualization operater and created a virtualization cluster.  Further we went ahead and configured an external bridge so that are virtual machines can be connected to the outside network.  Therefore we are now at the point where we can launch a virtual machine.  To do this we will use the following yaml file below.  Lets go ahead and create the file:

~~~bash
[lab-user@provision scripts]$ cat << EOF > ~/virtualmachine-fedora.yaml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  annotations:
    kubevirt.io/latest-observed-api-version: v1alpha3
    kubevirt.io/storage-observed-api-version: v1alpha3
    name.os.template.kubevirt.io/silverblue32: Fedora 31 or higher
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachines/fedora
  resourceVersion: '65643'
  name: fedora
  uid: 24c12216-ba66-49ec-beae-414ea4e7c06a
  creationTimestamp: '2020-10-17T21:50:00Z'
  generation: 4
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
            'f:flavor.template.kubevirt.io/medium': {}
            'f:app': {}
            .: {}
            'f:workload.template.kubevirt.io/desktop': {}
            'f:vm.kubevirt.io/template.revision': {}
            'f:vm.kubevirt.io/template': {}
        'f:spec':
          .: {}
          'f:dataVolumeTemplates': {}
          'f:running': {}
          'f:template':
            .: {}
            'f:metadata':
              .: {}
              'f:labels':
                .: {}
                'f:flavor.template.kubevirt.io/medium': {}
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
      time: '2020-10-17T21:50:00Z'
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
      time: '2020-10-17T21:52:54Z'
  namespace: default
  labels:
    app: fedora
    flavor.template.kubevirt.io/medium: 'true'
    os.template.kubevirt.io/silverblue32: 'true'
    vm.kubevirt.io/template: fedora-desktop-medium-v0.11.3
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
        name: fedora-rootdisk
      spec:
        pvc:
          accessModes:
            - ReadWriteMany
          resources:
            requests:
              storage: 10Gi
          storageClassName: ocs-storagecluster-ceph-rbd
          volumeMode: Block
        source:
          http:
            url: >-
              https://download.fedoraproject.org/pub/fedora/linux/releases/32/Cloud/x86_64/images/Fedora-Cloud-Base-32-1.6.x86_64.raw.xz
      status: {}
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        flavor.template.kubevirt.io/medium: 'true'
        kubevirt.io/domain: fedora
        kubevirt.io/size: medium
        os.template.kubevirt.io/silverblue32: 'true'
        vm.kubevirt.io/name: fedora
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
            memory: 4Gi
      evictionStrategy: LiveMigrate
      hostname: fedora32
      networks:
        - multus:
            networkName: brext
          name: nic-0
      terminationGracePeriodSeconds: 180
      volumes:
        - dataVolume:
            name: fedora-rootdisk
          name: rootdisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              name: default
              hostname: fedora32
              password: redhat
              chpasswd: {expire: False}
          name: cloudinitdisk
status:
  conditions:
    - lastProbeTime: null
      lastTransitionTime: '2020-10-17T21:52:54Z'
      status: 'True'
      type: Ready
  created: true
  ready: true
EOF
~~~

Lets take a look at the file we just created:

~~~bash 
[lab-user@provision scripts]$ cat ~/virtualmachine-fedora.yaml 
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  annotations:
    kubevirt.io/latest-observed-api-version: v1alpha3
    kubevirt.io/storage-observed-api-version: v1alpha3
    name.os.template.kubevirt.io/silverblue32: Fedora 31 or higher
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachines/fedora
  resourceVersion: '65643'
  name: fedora
  uid: 24c12216-ba66-49ec-beae-414ea4e7c06a
  creationTimestamp: '2020-10-17T21:50:00Z'
  generation: 4
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
            'f:flavor.template.kubevirt.io/medium': {}
            'f:app': {}
            .: {}
            'f:workload.template.kubevirt.io/desktop': {}
            'f:vm.kubevirt.io/template.revision': {}
            'f:vm.kubevirt.io/template': {}
        'f:spec':
          .: {}
          'f:dataVolumeTemplates': {}
          'f:running': {}
          'f:template':
            .: {}
            'f:metadata':
              .: {}
              'f:labels':
                .: {}
                'f:flavor.template.kubevirt.io/medium': {}
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
      time: '2020-10-17T21:50:00Z'
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
      time: '2020-10-17T21:52:54Z'
  namespace: default
  labels:
    app: fedora
    flavor.template.kubevirt.io/medium: 'true'
    os.template.kubevirt.io/silverblue32: 'true'
    vm.kubevirt.io/template: fedora-desktop-medium-v0.11.3
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
        name: fedora-rootdisk
      spec:
        pvc:
          accessModes:
            - ReadWriteMany
          resources:
            requests:
              storage: 10Gi
          storageClassName: ocs-storagecluster-ceph-rbd
          volumeMode: Block
        source:
          http:
            url: >-
              https://download.fedoraproject.org/pub/fedora/linux/releases/32/Cloud/x86_64/images/Fedora-Cloud-Base-32-1.6.x86_64.raw.xz
      status: {}
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        flavor.template.kubevirt.io/medium: 'true'
        kubevirt.io/domain: fedora
        kubevirt.io/size: medium
        os.template.kubevirt.io/silverblue32: 'true'
        vm.kubevirt.io/name: fedora
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
            memory: 4Gi
      evictionStrategy: LiveMigrate
      hostname: fedora32
      networks:
        - multus:
            networkName: brext
          name: nic-0
      terminationGracePeriodSeconds: 180
      volumes:
        - dataVolume:
            name: fedora-rootdisk
          name: rootdisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              name: default
              hostname: fedora32
              password: redhat
              chpasswd: {expire: False}
          name: cloudinitdisk
status:
  conditions:
    - lastProbeTime: null
      lastTransitionTime: '2020-10-17T21:52:54Z'
      status: 'True'
      type: Ready
  created: true
  ready: true
~~~

Now lets create the virtual machine:

~~~bash
[lab-user@provision scripts]$ oc create -f ~/virtualmachine-fedora.yaml 
virtualmachine.kubevirt.io/cirros created
~~~

If we run a oc get pods for the default namespace we will see an importer container.  Once it is running we can watch the logs:

~~~bash
[lab-user@provision ~]$ oc get pods
NAME                       READY   STATUS    RESTARTS   AGE
importer-fedora-rootdisk   0/1     Pending   0          2s
[lab-user@provision ~]$ oc get pods
NAME                       READY   STATUS    RESTARTS   AGE
importer-fedora-rootdisk   1/1     Running   0          18s
~~~

You can following the image importing process container which is pulling in the fedora image from the URL we had inside the virtualmachine-fedora.yaml file:

~~~bash
[lab-user@provision ~]$ oc logs importer-fedora-rootdisk -f
I1018 10:14:55.118650       1 importer.go:51] Starting importer
I1018 10:14:55.119845       1 importer.go:112] begin import process
I1018 10:14:55.722075       1 data-processor.go:277] Calculating available size
I1018 10:14:55.722969       1 data-processor.go:285] Checking out block volume size.
I1018 10:14:55.722979       1 data-processor.go:297] Request image size not empty.
I1018 10:14:55.722987       1 data-processor.go:302] Target size 10Gi.
I1018 10:14:55.825077       1 data-processor.go:206] New phase: TransferDataFile
I1018 10:14:55.826012       1 util.go:161] Writing data...
I1018 10:14:56.825462       1 prometheus.go:69] 0.14
I1018 10:14:57.825748       1 prometheus.go:69] 0.15
I1018 10:14:58.826019       1 prometheus.go:69] 1.00
I1018 10:14:59.826160       1 prometheus.go:69] 2.31
I1018 10:15:00.826360       1 prometheus.go:69] 3.77
I1018 10:15:01.826512       1 prometheus.go:69] 4.47
I1018 10:15:02.826838       1 prometheus.go:69] 5.33
I1018 10:15:03.827362       1 prometheus.go:69] 5.50
I1018 10:15:04.827521       1 prometheus.go:69] 5.75
I1018 10:15:05.827674       1 prometheus.go:69] 6.01
I1018 10:15:06.827881       1 prometheus.go:69] 6.03
I1018 10:15:07.828369       1 prometheus.go:69] 6.81
I1018 10:15:08.828523       1 prometheus.go:69] 7.84
I1018 10:15:09.828691       1 prometheus.go:69] 9.02
(...)
I1018 10:17:07.871219       1 prometheus.go:69] 100.00
I1018 10:17:08.872312       1 prometheus.go:69] 100.00
I1018 10:17:10.215935       1 data-processor.go:206] New phase: Resize
I1018 10:17:10.217033       1 data-processor.go:206] New phase: Complete
I1018 10:17:10.217151       1 importer.go:175] Import complete
~~~

Once the importer process has completed the virtual machine will then begin to startup and go to a running state.  Lets install virtctl before we proceed:

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

We can use virtctl to interact with the virtual machine much in the was we use the virsh command:

~~~bash
[lab-user@provision ~]$ virtctl -h
virtctl controls virtual machine related operations on your kubernetes cluster.

Available Commands:
  console      Connect to a console of a virtual machine instance.
  expose       Expose a virtual machine instance, virtual machine, or virtual machine instance replica set as a new service.
  help         Help about any command
  image-upload Upload a VM image to a DataVolume/PersistentVolumeClaim.
  migrate      Migrate a virtual machine.
  pause        Pause a virtual machine
  restart      Restart a virtual machine.
  start        Start a virtual machine.
  stop         Stop a virtual machine.
  unpause      Unpause a virtual machine
  version      Print the client and server version information.
  vnc          Open a vnc connection to a virtual machine instance.

Use "virtctl <command> --help" for more information about a given command.
Use "virtctl options" for a list of global command-line options (applies to all commands).
~~~

Lets go ahead and see what the status of our virtual machine by connecting to the console (you may need to hit enter a couple of times):

~~~bash
[lab-user@provision ~]$ virtctl console fedora
Successfully connected to fedora console. The escape sequence is ^]

fedora32 login: 
~~~

As defined by the cloud-init information in our virtual machine yaml file we used the passwd should be set to redhat for the fedora user.  Lets login:

~~~bash
fedora32 login: fedora
Password: 
[fedora@fedora32 ~]$ cat /etc/fedora-release
Fedora release 32 (Thirty Two)
~~~

Now lets see if we got a 172.22.0.0/24 network address.  If you reference back to the virtual machines yaml file you will notice we used the brext bridge interface as the interface our virtual machine should be connected to.  Thus we should have a 172.22.0.0/24 network address and access outside:

~~~bash
[fedora@fedora32 ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 46:84:6a:fb:ed:ee brd ff:ff:ff:ff:ff:ff
    altname enp1s0
    inet 172.22.0.29/24 brd 172.22.0.255 scope global dynamic noprefixroute eth0
       valid_lft 3176sec preferred_lft 3176sec
    inet6 fe80::4484:6aff:fefb:edee/64 scope link 
       valid_lft forever preferred_lft forever
~~~

Lets see if we can ping the gateway of 172.22.0.1:

~~~bash
[fedora@fedora32 ~]$ ping 172.22.0.1
PING 172.22.0.1 (172.22.0.1) 56(84) bytes of data.
64 bytes from 172.22.0.1: icmp_seq=1 ttl=64 time=1.63 ms
64 bytes from 172.22.0.1: icmp_seq=2 ttl=64 time=0.800 ms
64 bytes from 172.22.0.1: icmp_seq=3 ttl=64 time=0.848 ms
~~~

Looks like we have successful external network connectivity!

Now lets escape out of the console session and connect to the fedora instance via ssh from the provisioning host:

~~~bash
fedora32 login: [lab-user@provision ocp]$ ssh fedora@172.22.0.29
The authenticity of host '172.22.0.29 (172.22.0.29)' can't be established.
ECDSA key fingerprint is SHA256:KAqdT3NUhrat+tfEV8e9V/hvWL8v5CQVsbULKxCLZp8.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.22.0.29' (ECDSA) to the list of known hosts.
fedora@172.22.0.29's password: 
Last login: Sun Oct 18 15:31:04 2020
[systemd]
Failed Units: 1
  dnf-makecache.service
[fedora@fedora32 ~]$ 
[fedora@fedora32 ~]$ 
[fedora@fedora32 ~]$ exit
~~~

Now our external network in this lab is actually the provisioning network we created a bridge on so it really does not have full functional connectivity to the internet.  However we can add a few bits to confirm that indeed if our bridge was a truely routable network we would be able to access the internet.  To simulate this we need to add the following firewalld rule on the provisioning node:

~~~bash
[lab-user@provision ~]$ sudo firewall-cmd --direct --add-rule ipv4 nat POSTROUTING 0 -o baremetal -j MASQUERADE
success
~~~

Then back on our fedora virtual machine we need to add the following default gateway which is the IP of the provisioning nodes interface:

~~~bash
[fedora@fedora32 ~]$ sudo route add default gw 172.22.0.1
[fedora@fedora32 ~]$ ip route
default via 172.22.0.1 dev eth0 
172.22.0.0/24 dev eth0 proto kernel scope link src 172.22.0.29 metric 100
~~~

Now lets try to ping Google's nameserver:

~~~bash
[fedora@fedora32 ~]$ ping -c 4 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=116 time=2.81 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=116 time=1.89 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=116 time=2.16 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=116 time=1.91 ms

--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 1.886/2.192/2.807/0.370 ms
~~~

As you can see we were able to access the outside world!


**Success**, we're done! Congratulations... if you've made it this far you've deployed KNI from the ground up, deployed Ceph via Rook, Container Native Virtualisation (CNV), and tested the solution with pods and VM's via the CLI and the OpenShift dashboard! I'd like to ***thank you*** for attending this lab; I hope that it was a valuable use of your time and that your learnt a lot from doing it. Please do let us know if there's anything else we can do to support you! There's also a CNV-based lab here at Red Hat Tech Exchange if you're keen on exploring it further.
