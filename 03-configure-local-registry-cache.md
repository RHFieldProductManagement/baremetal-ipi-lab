 #**Configure Disconnected Registry and RHCOS httpd Cache**
 
 In this lab we will explore what has been a popular topic when it comes to OpenShift: the disconnected installation.  A disconnected installation is one where the master and worker nodes do not have access to the internet. Thus the Red Hat CoreOS images and the OpenShift pod images need to be hosted locally to support the given installation.
 
 Our current lab already has some of the components needed for a disconnected installation so lets first explore those.  The first component needed is the oc client.  If we type oc version at the command line we can see what version we have:
 
 ~~~bash
[lab-user@provision ~]$ cd scripts/
[lab-user@provision scripts]$ 
[lab-user@provision scripts]$ oc version
Client Version: 4.5.9
Server Version: 4.5.9
Kubernetes Version: v1.18.3+6c42de8
~~~

The output above shows us we have the 4.5.9 client for oc.  This will determine what version of the OpenShift cluster we will deploy.  Lets go ahead and set the environment variable of VERSION to that version now:

~~~bash
[lab-user@provision scripts]$ export VERSION=4.5.9
[lab-user@provision scripts]$ echo $VERSION
4.5.9
~~~

Now lets examine the version of the openshift-baremetal-install binary version.  The commit number is important as that will be used to determine what version of the RHCOS image is pulled down later on in this section of the lab.

~~~bash
[lab-user@provision scripts]$ ./openshift-baremetal-install version
./openshift-baremetal-install 4.5.9
built from commit 0d5c871ce7d03f3d03ab4371dc39916a5415cf5c
release image quay.io/openshift-release-dev/ocp-release@sha256:7ad540594e2a667300dd2584fe2ede2c1a0b814ee6a62f60809d87ab564f4425
~~~

Now that we have examined the oc and openshift-baremetal-install binaries we are ready to build our private registry and httpd cache on the provisioning node.   In a production environment this registry and httpd cache could be anywhere within the organization but for this lab we will keep it simple and on the provisioning node.

The first step is to install podman and httpd which will also pull in some additional dependencies:

~~~bash
[lab-user@provision scripts]$ sudo yum -y install podman httpd httpd-tools
Updating Subscription Management repositories.
Red Hat Enterprise Linux 8 for x86_64 - BaseOS (RPMs)                                                                                                                              3.9 kB/s | 2.4 kB     00:00 
(...)
Installed:
  apr-1.6.3-9.el8.x86_64                                                   apr-util-1.6.1-6.el8.x86_64                                         apr-util-bdb-1.6.1-6.el8.x86_64                                   
  apr-util-openssl-1.6.1-6.el8.x86_64                                      conmon-2:2.0.6-1.module+el8.2.0+6368+cf16aa14.x86_64                container-selinux-2:2.124.0-1.module+el8.2.0+6368+cf16aa14.noarch 
  containernetworking-plugins-0.8.3-5.module+el8.2.0+6368+cf16aa14.x86_64  containers-common-1:0.1.40-11.module+el8.2.0+6374+67f43e89.x86_64   criu-3.12-9.module+el8.2.0+6368+cf16aa14.x86_64                   
  fuse-overlayfs-0.7.2-5.module+el8.2.0+6368+cf16aa14.x86_64               fuse3-libs-3.2.1-12.el8.x86_64                                      httpd-2.4.37-21.module+el8.2.0+5008+cca404a3.x86_64               
  httpd-filesystem-2.4.37-21.module+el8.2.0+5008+cca404a3.noarch           httpd-tools-2.4.37-21.module+el8.2.0+5008+cca404a3.x86_64           libnet-1.1.6-15.el8.x86_64                                        
  libvarlink-18-3.el8.x86_64                                               mailcap-2.1.48-3.el8.noarch                                         mod_http2-1.11.3-3.module+el8.2.0+4377+dc421495.x86_64            
  podman-1.6.4-11.module+el8.2.0+6368+cf16aa14.x86_64                      protobuf-c-1.3.0-4.el8.x86_64                                       redhat-logos-httpd-81.1-1.el8.noarch                              
  runc-1.0.0-65.rc10.module+el8.2.0+6368+cf16aa14.x86_64                   slirp4netns-0.4.2-3.git21fdece.module+el8.2.0+6368+cf16aa14.x86_64 

Complete!
~~~

Now lets create the directories you'll need to run the registry. These directories will be mounted in the container runtime environment for the registry.

~~~bash
[lab-user@provision scripts]$ sudo mkdir -p /nfs/registry/{auth,certs,data}
[lab-user@provision scripts]$ 
~~~

We also need to create a self signed certificate for the registry:

~~~bash
[lab-user@provision scripts]$ sudo openssl req -newkey rsa:4096 -nodes -sha256 -keyout /nfs/registry/certs/domain.key -x509 -days 365 -out /nfs/registry/certs/domain.crt -subj "/C=US/ST=NorthCarolina/L=Raleigh/O=Red Hat/OU=Marketing/CN=provision.$GUID.dynamic.opentlc.com"
Generating a RSA private key
......................................................................................................................................................................................................................................................................++++
.............................................................................................................................................................................................................................................................++++
writing new private key to '/nfs/registry/certs/domain.key'
-----
~~~

Once the certificate has been created lets copy it into our home directory and also into the trust anchors on the provisioning node.  We will also need to run the update-ca-trust command:

~~~bash
[lab-user@provision scripts]$ sudo cp /nfs/registry/certs/domain.crt $HOME/scripts/domain.crt
[lab-user@provision scripts]$ sudo chown lab-user $HOME/scripts/domain.crt
[lab-user@provision scripts]$ sudo cp /nfs/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/
[lab-user@provision scripts]$ sudo update-ca-trust extract
~~~

Our registry will need a simple authentication mechanism so we will use htpasswd.  Note that when you try to authenticate to your registry the password being passed has to be in bcrypt format.

~~~bash
[lab-user@provision scripts]$ sudo htpasswd -bBc /nfs/registry/auth/htpasswd dummy dummy
Adding password for user dummy
~~~

Now that we have a directy structure, certificate and a user configured for authentication we can go ahead and create the registry pod.  The command below will pull down the pod and mount the appropriate directory mount points we created earlier.

~~~bash
[lab-user@provision scripts]$ sudo podman create --name poc-registry --net host -p 5000:5000 -v /nfs/registry/data:/var/lib/registry:z -v /nfs/registry/auth:/auth:z -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry" -e "REGISTRY_HTTP_SECRET=ALongRandomSecretForRegistry" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd -v /nfs/registry/certs:/certs:z -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key docker.io/library/registry:2
Trying to pull docker.io/library/registry:2...
Getting image source signatures
Copying blob cbdbe7a5bc2a done
Copying blob c1cc712bcecd done
Copying blob 47112e65547d done
Copying blob 3db6272dcbfa done
Copying blob 46bcb632e506 done
Copying config 2d4f4b5309 done
Writing manifest to image destination
Storing signatures
be06131e5dc4b98a1f55fdefc6afa6989cfbc8d878b6d65cf40426e96e2bede1
~~~

Once pod creation is complete we can next start the pod:

~~~bash
[lab-user@provision scripts]$ sudo podman start poc-registry
poc-registry
~~~

Finally lets verify the pod is up and running via the podman command:

~~~bash
[lab-user@provision scripts]$ sudo podman ps
CONTAINER ID  IMAGE                         COMMAND               CREATED        STATUS             PORTS  NAMES
be06131e5dc4  docker.io/library/registry:2  /etc/docker/regis...  2 minutes ago  Up 39 seconds ago         poc-registry
~~~

We can further validate the registry is functional by using a curl command and passing the user/password to the registry URL.  Note here I do not have to use a bcrypt formatted password.

~~~bash
[lab-user@provision scripts]$ curl -u dummy:dummy -k https://provision.$GUID.dynamic.opentlc.com:5000/v2/_catalog
{"repositories":[]}
~~~

Now that our registry pod is up and we have validated it working lets configure the httpd cache pod which will store our Red Hat CoreOS images locally.  The first step is to create some directory structures and add the appropriate permissions:

~~~bash
[lab-user@provision scripts]$ export IRONIC_DATA_DIR=/nfs/ocp/ironic
[lab-user@provision scripts]$ export IRONIC_IMAGES_DIR="${IRONIC_DATA_DIR}/html/images"
[lab-user@provision scripts]$ export IRONIC_IMAGE=quay.io/metal3-io/ironic:master
[lab-user@provision scripts]$ sudo mkdir -p $IRONIC_IMAGES_DIR
[lab-user@provision scripts]$ sudo chown -R "${USER}:${USER}" "$IRONIC_DATA_DIR"
[lab-user@provision scripts]$ sudo find $IRONIC_DATA_DIR -type d -print0 | xargs -0 chmod 755
[lab-user@provision scripts]$ sudo chmod -R +r $IRONIC_DATA_DIR
~~~

With the directory structures in place we can now create the caching pod and in our case we are using the the ironic pod that already exists in quay.io.

~~~bash
[lab-user@provision scripts]$ sudo podman pod create -n ironic-pod
12385a4f6f8cb912e7733b725c2b488de4e21aef049552efd21afc28dd647014
[lab-user@provision scripts]$ sudo podman run -d --net host --privileged --name httpd --pod ironic-pod -v $IRONIC_DATA_DIR:/shared --entrypoint /bin/runhttpd ${IRONIC_IMAGE}
Trying to pull quay.io/metal3-io/ironic:master...
Getting image source signatures
Copying blob 3c72a8ed6814 done
Copying blob dedbfd2c2275 done
Copying blob c7075fe6e7e3 done
Copying blob b9d82df42627 done
Copying blob 2f2cf2a5ca6f done
Copying blob e3e9e5cd6698 done
Copying blob 154a03d6108d done
Copying blob eb117e61d6ae done
Copying blob e6725534ffd1 done
Copying blob e90326c4db9a done
Copying blob 3781fb791002 done
Copying blob fed77fc47bdf done
Copying blob 8d14d6939957 done
Copying blob 178237ba390f done
Copying blob 3de95aba020f done
Copying blob 558fb08cb05e done
Copying blob 301d166a7a5e done
Copying blob 50adcfd6cc77 done
Copying blob 80bc06ff32a9 done
Copying blob 3c09f26b5dc9 done
Copying blob a3a07a6a652f done
Copying blob 6d2ffbec6c34 done
Copying blob db435f5910cb done
Copying config 3733498f02 done
Writing manifest to image destination
Storing signatures
f069949f68fa147206d154417a22c20c49983f0c5b79e9c06d56750e9d3f470d
~~~

Because we ran the create command and then a run command after there is no need to actually use podman to start the httpd pod.  We can see that if we look at the current running pods on the provisioning node:

~~~bash
[lab-user@provision scripts]$ sudo podman ps
CONTAINER ID  IMAGE                            COMMAND               CREATED         STATUS             PORTS  NAMES
f069949f68fa  quay.io/metal3-io/ironic:master                        8 seconds ago   Up 7 seconds ago          httpd
be06131e5dc4  docker.io/library/registry:2     /etc/docker/regis...  22 minutes ago  Up 20 minutes ago         poc-registry
~~~

Further we can test that our httpd cache is operational by using the curl command.  If you get a 301 code that is normal since we have yet to actually place any images in the cache.

~~~bash
[lab-user@provision scripts]$ curl http://provision.$GUID.dynamic.opentlc.com/images
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>301 Moved Permanently</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="http://provision.schmaustech.dynamic.opentlc.com/images/">here</a>.</p>
</body></html>
~~~~

We are almost ready to do some downloading of images but we still have a few items to tend to.  First we need to generate a bcrypt password from our username and password we set on the registry.  We can do this by piping them into base64 and capturing the output:

~~~bash
[lab-user@provision scripts]$ echo -n 'dummy:dummy' | base64 -w0
ZHVtbXk6ZHVtbXk=
~~~

Next we will want to take the output above and craft it into a registry secret text file:

~~~bash
[lab-user@provision scripts]$   cat <<EOF >> ~/reg-secret.txt
> "provision.$GUID.dynamic.opentlc.com:5000": {
>   "email": "dummy@redhat.com",
>   "auth": "ZHVtbXk6ZHVtbXk="
> }
> EOF
~~~

Now we can take our existing lab pull secret and our registry pull secret and merge them.  Below we will backup the existing pull secret and then add our registry pull secret to the file.  Further we will add our pull secret and registry cert, as a trust bundle, to the existing install-config.yaml.

~~~bash
[lab-user@provision scripts]$ export PULLSECRET=$HOME/pull-secret.json
[lab-user@provision scripts]$ cp $PULLSECRET $PULLSECRET.orig
[lab-user@provision scripts]$ cat $PULLSECRET | jq ".auths += {`cat ~/reg-secret.txt`}" > $PULLSECRET
[lab-user@provision scripts]$ cat $PULLSECRET | tr -d '[:space:]' > tmp-secret
[lab-user@provision scripts]$ mv -f tmp-secret $PULLSECRET
[lab-user@provision scripts]$ rm -f ~/reg-secret.txt
[lab-user@provision scripts]$ sed -i -e 's/^/  /' $(pwd)/domain.crt
[lab-user@provision scripts]$ echo "additionalTrustBundle: |" >> $HOME/scripts/install-config.yaml
[lab-user@provision scripts]$ cat $(pwd)/domain.crt >> $HOME/scripts/install-config.yaml
[lab-user@provision scripts]$ sed -i "s/pullSecret:.*/pullSecret: \'$(cat $PULLSECRET)\'/g" $HOME/scripts/install-config.yaml
~~~

If you cat out the install-config.yaml you should be able to see the changes we made.

~~~bash
[lab-user@provision scripts]$ cat install-config.yaml
apiVersion: v1
baseDomain: dynamic.opentlc.com
(...)
"},"provision.schmaustech.dynamic.opentlc.com:5000":{"email":"dummy@redhat.com","auth":"ZHVtbXk6ZHVtbXk="}}}'
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  MIIGDzCCA/egAwIBAgIUc3tgxZl2g92XdCUX15hWMAIGi10wDQYJKoZIhvcNAQEL
  BQAwgZYxCzAJBgNVBAYTAlVTMRYwFAYDVQQIDA1Ob3J0aENhcm9saW5hMRAwDgYD
  VQQHDAdSYWxlaWdoMRAwDgYDVQQKDAdSZWQgSGF0MRIwEAYDVQQLDAlNYXJrZXRp
  bmcxNzA1BgNVBAMMLnByb3Zpc2lvbi5zY2htYXVzdGVjaC5zdHVkZW50cy5vc3Au
  b3BlbnRsYy5jb20wHhcNMjAxMDA1MTM0OTAzWhcNMjExMDA1MTM0OTAzWjCBljEL
  MAkGA1UEBhMCVVMxFjAUBgNVBAgMDU5vcnRoQ2Fyb2xpbmExEDAOBgNVBAcMB1Jh
  bGVpZ2gxEDAOBgNVBAoMB1JlZCBIYXQxEjAQBgNVBAsMCU1hcmtldGluZzE3MDUG
  A1UEAwwucHJvdmlzaW9uLnNjaG1hdXN0ZWNoLnN0dWRlbnRzLm9zcC5vcGVudGxj
  LmNvbTCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCCAgoCggIBAMvffu+qXR0r3Yxg
  Z1tUKYejJTmEXf7e4JDlKWyijeu8buDJD0T544gBtWDbEwpho7lsnRgC7w5Peasc
  DpOQAqI980vQp8tAnS9ncJVroUAtNtf3WLLVpoEPbNTyRdZ2clEh17KcnJQ4Hsjd
  mMiRNMLzmjBocAXeA2mGkjm2ZN/+fkaC2Zk1DtcPPuF7+apNRk9dizqYawupwgrF
  zSjFitvf1IC49NtO5b01VWW3056HX+bx8KkGAAMNvqaRlz703HWEeplfsEkyVvTL
  SOF2BJIbS1HxYZ92qnwIVjzgdx8eZPV954pDvQovEXJExShn9mDEZWuQDcwnwdyU
  o+zgvzp1dFm9y6iC1u+8eG5wnmoJRkFyxkE3Uoysj2yMcSNGUK8z9O2rBulA8rPC
  IO4oaizL102wUHj6+ESvbYm5Gjzj/trKuhEtCXYmtyndHe1PsKRmUEq8dZAJBrXY
  axasroyODSIN6g6wSNSyS490wfu4QZnuEb1X9qXNsvNOgGRwrCEodyAiwCMvVNMw
  eDA2XAukNktOUVmzrQiupn37lGVhpl47ssmPW5EKWI9SNehz09x16ZVlGRs2ojQU
  XbutiswxseFm8Qn9teBKLqR2HuOAZb5xS9EDesocwoGRnenmqP+jYt8ifq4ajOEV
  nJz8oGpIt9gLWaay0fnIzfG08KXHAgMBAAGjUzBRMB0GA1UdDgQWBBTJDjq5gFPw
  5oqaDPOci3iikL61GTAfBgNVHSMEGDAWgBTJDjq5gFPw5oqaDPOci3iikL61GTAP
  BgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3DQEBCwUAA4ICAQDBHCZeAKgkcGAGZOfS
  F3ohdYj50MeN/lVtESbvUlirMGEs2f932YkY1oF8ulFy2n3EZftTTpUo2/tDKik7
  3rZs/cCD8KrPnHAdSJGny7ud27w85DM+dFTwxuIjdHAXdMhoOKvV+lSkziW9Ltmg
  p7MbOei2nqpxTpX42DfqqC2ZRZ1KyyQ8EClqTlYh3iozwyp1VwpHBnQkFnZLfLjc
  cHcbayrEgxN7TxUJYqHUP90A7guHA1OfWSSduNN1b8aFACegOtb9MFTRjbIrbNw4
  R1t5D5TsMc8RfIETHE+9xb1HLdojnXQA8Hwp3myVL7PNr6tKu01hKvbhhhykACFz
  KtVFLCdv3EF/dZJHahQSJksThFY6Jaeyj7rE6OJ1lJbB/RMGdV/3l7kyDbs7A/mf
  XKt6I4WoymEDcC7dlcif4sQHMWCKwHMtT8pen04T48CGb/5GLGBVwkx6qEynOE3S
  KukJj2o1QZJOSi5KdSfGeILAUHW1eOOWank9l1SIS5OIaNBGkSmem6J9heGy6ulv
  5IU9ahv5IMoJS8wJgkgTMc5B2B/Mbv2dL+kthbemyyPCdN62QtlvhkLCiOU4niI3
  JdXFoPLSZ5nEb+Y/XB+WVaaz+j8CtGlDcwbGr4BFlakHluVxfK2sDN5n0NOLcQlw
  Z2sCrU3XLJDME6tPetuPiBX/tA==
  -----END CERTIFICATE-----
~~~

Finally at this point we can sync down the pod images from quay.io to our local registry.  To do this we need to take a few steps below:

~~~bash
[lab-user@provision scripts]$ export UPSTREAM_REPO="quay.io/openshift-release-dev/ocp-release:$VERSION-x86_64"
[lab-user@provision scripts]$ export PULLSECRET=$HOME/pull-secret.json
[lab-user@provision scripts]$ export LOCAL_REG="provision.$GUID.dynamic.opentlc.com:5000"
[lab-user@provision scripts]$ export LOCAL_REPO='ocp4/openshift4'
~~~

So what did we do above?  We ended using the version variable we set earlier to help us set the upstream registry and repository for our 4.5.9 release.  Further we set a pull secet variable and then our local registry and local repository variables.

Now we can actually execute the mirroring:

~~~bash
[lab-user@provision scripts]$ oc adm release mirror -a $PULLSECRET --from=$UPSTREAM_REPO --to-release-image=$LOCAL_REG/$LOCAL_REPO:$VERSION --to=$LOCAL_REG/$LOCAL_REPO
info: Mirroring 110 images to provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4 ...
provision.schmaustech.dynamic.opentlc.com:5000/
  ocp4/openshift4
    manifests:
      sha256:00edb6c1dae03e1870e1819b4a8d29b655fb6fc40a396a0db2d7c8a20bd8ab8d -> 4.5.9-local-storage-static-provisioner
      sha256:0259aa5845ce43114c63d59cedeb71c9aa5781c0a6154fe5af8e3cb7bfcfa304 -> 4.5.9-machine-api-operator
      sha256:07f11763953a2293bac5d662b6bd49c883111ba324599c6b6b28e9f9f74112be -> 4.5.9-cluster-kube-storage-version-migrator-operator
(...)
sha256:15be0e6de6e0d7bec726611f1dcecd162325ee57b993e0d886e70c25a1faacc3 provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4:4.5.9-openshift-controller-manager
sha256:bc6c8fd4358d3a46f8df4d81cd424e8778b344c368e6855ed45492815c581438 provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4:4.5.9-hyperkube
sha256:bcd6cd1559b62e4a8031cf0e1676e25585845022d240ac3d927ea47a93469597 provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4:4.5.9-machine-config-operator
sha256:b05f9e685b3f20f96fa952c7c31b2bfcf96643e141ae961ed355684d2d209310 provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4:4.5.9-baremetal-installer
info: Mirroring completed in 1.48s (0B/s)

Success
Update image:  provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4:4.5.9
Mirror prefix: provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4

To use the new mirrored repository to install, add the following section to the install-config.yaml:

imageContentSources:
- mirrors:
  - provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4
  source: registry.svc.ci.openshift.org/ocp/release


To use the new mirrored repository for upgrades, use the following to create an ImageContentSourcePolicy:

apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: example
spec:
  repositoryDigestMirrors:
  - mirrors:
    - provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
  - mirrors:
    - provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4
    source: registry.svc.ci.openshift.org/ocp/release
~~~

The above command will take some time but once complete should have mirrored all the required images from the remote registry to our local registry.   Once key piece of output above is the imageContentSources.  That section of the output is needed for the install-config.yaml file that is used for our OpenShift deployment.   If you look at the current ~/scripts/install-config.yaml you will notice those lines have already been added to the config for you.

Now that we have the images synced down we can move on to syncing the RHCOS images needed for which their are two: an RHCOS qemu image and RHCOS openstack image.  The RHCOS qemu image is the image used for the bootstrap virtual machines that is created on the provisioning host during the initial phases of the deployment process.  The openstack image is the one used to image the master and worker nodes during the deployment process.

To capture this image we have to set a few different environment variables to ensure we download the correct RHCOS image for the version release we are using.  Again in this labs case we are installing 4.5.9.  Lets set a few variables:

~~~bash
[lab-user@provision scripts]$ OPENSHIFT_INSTALLER=$HOME/scripts/openshift-baremetal-install
[lab-user@provision scripts]$ IRONIC_DATA_DIR=/nfs/ocp/ironic
[lab-user@provision scripts]$ OPENSHIFT_INSTALL_COMMIT=$($OPENSHIFT_INSTALLER version | grep commit | cut -d' ' -f4)
[lab-user@provision scripts]$ OPENSHIFT_INSTALLER_MACHINE_OS=${OPENSHIFT_INSTALLER_MACHINE_OS:-https://raw.githubusercontent.com/openshift/installer/$OPENSHIFT_INSTALL_COMMIT/data/data/rhcos.json}

[lab-user@provision scripts]$ MACHINE_OS_IMAGE_JSON=$(curl "${OPENSHIFT_INSTALLER_MACHINE_OS}")
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  5334  100  5334    0     0  17374      0 --:--:-- --:--:-- --:--:-- 17431

[lab-user@provision scripts]$ MACHINE_OS_INSTALLER_IMAGE_URL=$(echo "${MACHINE_OS_IMAGE_JSON}" | jq -r '.baseURI + .images.openstack.path')
[lab-user@provision scripts]$ MACHINE_OS_INSTALLER_IMAGE_SHA256=$(echo "${MACHINE_OS_IMAGE_JSON}" | jq -r '.images.openstack.sha256')
[lab-user@provision scripts]$ MACHINE_OS_IMAGE_URL=${MACHINE_OS_IMAGE_URL:-${MACHINE_OS_INSTALLER_IMAGE_URL}}
[lab-user@provision scripts]$ MACHINE_OS_IMAGE_NAME=$(basename ${MACHINE_OS_IMAGE_URL})
[lab-user@provision scripts]$ MACHINE_OS_IMAGE_SHA256=${MACHINE_OS_IMAGE_SHA256:-${MACHINE_OS_INSTALLER_IMAGE_SHA256}}
[lab-user@provision scripts]$ MACHINE_OS_INSTALLER_BOOTSTRAP_IMAGE_URL=$(echo "${MACHINE_OS_IMAGE_JSON}" | jq -r '.baseURI + .images.qemu.path')
[lab-user@provision scripts]$ MACHINE_OS_INSTALLER_BOOTSTRAP_IMAGE_SHA256=$(echo "${MACHINE_OS_IMAGE_JSON}" | jq -r '.images.qemu.sha256')
[lab-user@provision scripts]$ MACHINE_OS_BOOTSTRAP_IMAGE_URL=${MACHINE_OS_BOOTSTRAP_IMAGE_URL:-${MACHINE_OS_INSTALLER_BOOTSTRAP_IMAGE_URL}}
[lab-user@provision scripts]$ MACHINE_OS_BOOTSTRAP_IMAGE_NAME=$(basename ${MACHINE_OS_BOOTSTRAP_IMAGE_URL})
[lab-user@provision scripts]$ MACHINE_OS_BOOTSTRAP_IMAGE_SHA256=${MACHINE_OS_BOOTSTRAP_IMAGE_SHA256:-${MACHINE_OS_INSTALLER_BOOTSTRAP_IMAGE_SHA256}}
[lab-user@provision scripts]$ MACHINE_OS_INSTALLER_BOOTSTRAP_IMAGE_UNCOMPRESSED_SHA256=$(echo "${MACHINE_OS_IMAGE_JSON}" | jq -r '.images.qemu["uncompressed-sha256"]')
[lab-user@provision scripts]$ MACHINE_OS_BOOTSTRAP_IMAGE_UNCOMPRESSED_SHA256=${MACHINE_OS_BOOTSTRAP_IMAGE_UNCOMPRESSED_SHA256:-${MACHINE_OS_INSTALLER_BOOTSTRAP_IMAGE_UNCOMPRESSED_SHA256}}
~~~
  
Above we are doing quite a bit but its all in an effort to derive the right RHCOS image for both the bootstrap and installer image.  We first have to gather the commit string from the installer version.  Then we have to grab the rhcos json content with that commit information.  Next we pull out the appropriate image and sha256 for two RHCOS images and finally we set two sets of variables for those two images so we can pull them down below:
  
~~~bash
[lab-user@provision scripts]$ CACHED_MACHINE_OS_IMAGE="${IRONIC_DATA_DIR}/html/images/${MACHINE_OS_IMAGE_NAME}"
[lab-user@provision scripts]$ curl -g --insecure -L -o "${CACHED_MACHINE_OS_IMAGE}" "${MACHINE_OS_IMAGE_URL}"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   161  100   161    0     0   1319      0 --:--:-- --:--:-- --:--:--  1319
100  855M  100  855M    0     0  52.0M      0  0:00:16  0:00:16 --:--:-- 53.6M

[lab-user@provision scripts]$ echo "${MACHINE_OS_IMAGE_SHA256} ${CACHED_MACHINE_OS_IMAGE}" | tee ${CACHED_MACHINE_OS_IMAGE}.sha256sum
359e7c3560fdd91e64cd0d8df6a172722b10e777aef38673af6246f14838ab1a /nfs/ocp/ironic/html/images/rhcos-45.82.202008010929-0-openstack.x86_64.qcow2.gz
[lab-user@provision scripts]$ sha256sum --strict --check ${CACHED_MACHINE_OS_IMAGE}.sha256sum
/nfs/ocp/ironic/html/images/rhcos-45.82.202008010929-0-openstack.x86_64.qcow2.gz: OK


[lab-user@provision scripts]$ CACHED_MACHINE_OS_BOOTSTRAP_IMAGE="${IRONIC_DATA_DIR}/html/images/${MACHINE_OS_BOOTSTRAP_IMAGE_NAME}"
[lab-user@provision scripts]$ curl -g --insecure -L -o "${CACHED_MACHINE_OS_BOOTSTRAP_IMAGE}" "${MACHINE_OS_BOOTSTRAP_IMAGE_URL}"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   161  100   161    0     0   2012      0 --:--:-- --:--:-- --:--:--  2012
100  857M  100  857M    0     0  40.5M      0  0:00:21  0:00:21 --:--:-- 39.1M

[lab-user@provision scripts]$ echo "${MACHINE_OS_BOOTSTRAP_IMAGE_SHA256} ${CACHED_MACHINE_OS_BOOTSTRAP_IMAGE}" | tee ${CACHED_MACHINE_OS_BOOTSTRAP_IMAGE}.sha256sum
80ab9b70566c50a7e0b5e62626e5ba391a5f87ac23ea17e5d7376dcc1e2d39ce /nfs/ocp/ironic/html/images/rhcos-45.82.202008010929-0-qemu.x86_64.qcow2.gz
[lab-user@provision scripts]$ sha256sum --strict --check ${CACHED_MACHINE_OS_BOOTSTRAP_IMAGE}.sha256sum
/nfs/ocp/ironic/html/images/rhcos-45.82.202008010929-0-qemu.x86_64.qcow2.gz: OK


[lab-user@provision scripts]$ RHCOS_QEMU_IMAGE=$MACHINE_OS_BOOTSTRAP_IMAGE_NAME?sha256=$MACHINE_OS_INSTALLER_BOOTSTRAP_IMAGE_UNCOMPRESSED_SHA256
[lab-user@provision scripts]$ RHCOS_OPENSTACK_IMAGE=$MACHINE_OS_IMAGE_NAME?sha256=$MACHINE_OS_IMAGE_SHA256
[lab-user@provision scripts]$ sed -i "s/RHCOS_QEMU_IMAGE/$RHCOS_QEMU_IMAGE/g" $HOME/scripts/install-config.yaml
[lab-user@provision scripts]$ sed -i "s/RHCOS_OPENSTACK_IMAGE/$RHCOS_OPENSTACK_IMAGE/g" $HOME/scripts/install-config.yaml
~~~
 
Once the above commands have been run they should have downloaded two images: a RHCOS bootstrap qemu and a RHCOS openstack image.   We can confirm this by doing a directory listed on the $CACHED_MACHINE_OS_IMAGE and $CACHED_MACHINE_OS_BOOTSTRAP_IMAGE variables we set in the previous commands:

~~~bash
[lab-user@provision scripts]$ ls -l $CACHED_MACHINE_OS_IMAGE
-rw-rw-r--. 1 lab-user lab-user 896764070 Oct  5 11:39 /nfs/ocp/ironic/html/images/rhcos-45.82.202008010929-0-openstack.x86_64.qcow2.gz
[lab-user@provision scripts]$ ls -l $CACHED_MACHINE_OS_BOOTSTRAP_IMAGE
-rw-rw-r--. 1 lab-user lab-user 898670890 Oct  5 11:40 /nfs/ocp/ironic/html/images/rhcos-45.82.202008010929-0-qemu.x86_64.qcow2.gz
~~~

We can see the images are there.  We can further show they are accessible from our httpd cache by manually curling one of them (the Bootstrap image in this example):

~~~bash
[lab-user@provision scripts]$ curl http://provision.$GUID.dynamic.opentlc.com/images/rhcos-45.82.202008010929-0-qemu.x86_64.qcow2.gz -o test.qcow2
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  857M  100  857M    0     0   338M      0  0:00:02  0:00:02 --:--:--  338M
~~~

We can see we have a full sized image but lets remove it to save space:

~~~bash
[lab-user@provision scripts]$ ls -l test.qcow2
-rw-rw-r--. 1 lab-user lab-user 898670890 Oct  5 13:15 test.qcow2
[lab-user@provision scripts]$ rm test.qcow2
~~~

At the end of all of those commands we also ran two sed commands to update the install-config.yaml file with the appropriate paths for the bootstrap and cluster RHCOS images:

~~~bash
[lab-user@provision scripts]$ grep qcow install-config.yaml
    bootstrapOSImage: http://10.20.0.2/images/rhcos-45.82.202008010929-0-qemu.x86_64.qcow2.gz?sha256=c9e2698d0f3bcc48b7c66d7db901266abf27ebd7474b6719992de2d8db96995a
    clusterOSImage: http://10.20.0.2/images/rhcos-45.82.202008010929-0-openstack.x86_64.qcow2.gz?sha256=359e7c3560fdd91e64cd0d8df6a172722b10e777aef38673af6246f14838ab1a
~~~

As you can see it is rather easy to build a local registry and httpd cache for the pod images and RHCOS images.  In the next lab we will leverage this content with a deployment of OpenShift!
