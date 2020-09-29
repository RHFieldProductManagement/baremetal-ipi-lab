 #**Configure Disconnected Registry and RHCOS httpd Cache**
 
 In this lab we will explore what has been a popular topic when it comes to OpenShift: the disconnected installation.  A disconnected installation is one where the master and worker nodes do not have access to the internet. Thus the Red Hat CoreOS images and the OpenShift pod images need to be hosted locally to support the given installation.
 
 Our current lab already has some of the components needed for a disconnected installation so lets first explore those.  The first component needed is the oc client.  If we type oc version at the command line we can see what version we have:
 
 ~~~bash
 [cloud-user@provision scripts]$ oc version
Client Version: 4.5.9
Server Version: 4.5.9
Kubernetes Version: v1.18.3+6c42de8
~~~

The output above shows us we have the 4.5.9 client for oc.  This will determine what version of the OpenShift cluster we will deploy.  Lets go ahead and set the environment variable of VERSION to that version now:

~~~bash
[cloud-user@provision scripts]$ export VERSION=4.5.9
[cloud-user@provision scripts]$ echo $VERSION
4.5.9
~~~

Lets go ahead now and issue a sudo podman ps command to see what containers if any we have running on our provisioning node:

~~~bash
[cloud-user@provision scripts]$ sudo podman ps
CONTAINER ID  IMAGE                            COMMAND               CREATED       STATUS           PORTS  NAMES
9210ef6bc5f2  quay.io/metal3-io/ironic:master                        18 hours ago  Up 18 hours ago         httpd
ce5a09cb9115  docker.io/library/registry:2     /etc/docker/regis...  18 hours ago  Up 18 hours ago         poc-registry
~~~

As you can see from the output above we have two pods running: httpd and poc-registry.  The httpd pod is the one that will serve up the RHCOS images we are going to pull down and sync to this host.  The poc-registry is the local registry that will contain the pod images for the OpenShift installation locally.

Lets take a quick peek at each pod so we can see where the data is stored.  On both pods run a sudo podman inspect with the pod name:

~~~bash
[cloud-user@provision scripts]$ sudo podman inspect httpd
[
    {
        "Id": "9210ef6bc5f2c21a26e0963b4d57b779a1b0f6ddb5a875377aa24d6f3a3ded2a",
        "Created": "2020-09-28T15:26:33.16424928-04:00",
        "Path": "/bin/runhttpd",
        "Args": [
            "/bin/runhttpd"
        ],
(...)
        "Mounts": [
            {
                "Type": "bind",
                "Name": "",
                "Source": "/nfs/ocp/ironic",
                "Destination": "/shared",
                "Driver": "",
                "Mode": "",
                "Options": [
                    "rbind"
                ],
                "RW": true,
                "Propagation": "rprivate"
            }
        ],
(...)

[cloud-user@provision scripts]$ sudo podman inspect poc-registry
[
    {
        "Id": "ce5a09cb911596d54b62affed45d47f98b5d2b255cc0f79884ccbc873c050858",
        "Created": "2020-09-28T15:26:03.21874377-04:00",
        "Path": "/entrypoint.sh",
        "Args": [
            "/etc/docker/registry/config.yml"
        ],
(...)
        "Mounts": [
            {
                "Type": "bind",
                "Name": "",
                "Source": "/nfs/registry/certs",
                "Destination": "/certs",
                "Driver": "",
                "Mode": "",
                "Options": [
                    "rbind"
                ],
                "RW": true,
                "Propagation": "rprivate"
            },
            {
                "Type": "bind",
                "Name": "",
                "Source": "/nfs/registry/data",
                "Destination": "/var/lib/registry",
                "Driver": "",
                "Mode": "",
                "Options": [
                    "rbind"
                ],
                "RW": true,
                "Propagation": "rprivate"
            },
            {
                "Type": "bind",
                "Name": "",
                "Source": "/nfs/registry/auth",
                "Destination": "/auth",
                "Driver": "",
                "Mode": "",
                "Options": [
                    "rbind"
                ],
                "RW": true,
                "Propagation": "rprivate"
            }
        ],
(...)
~~~




Finally I want to point out that we created a domain.crt to be used for the poc-registry above.  A copy of that cert was placed into the scripts directory.  Lets go ahead and take a look at that cert:

~~~bash
[cloud-user@provision scripts]$ ls -l ~/scripts/domain.crt 
-rw-r--r--. 1 cloud-user cloud-user 2233 Sep 28 15:28 /home/cloud-user/scripts/domain.crt
[cloud-user@provision scripts]$ cat ~/scripts/domain.crt
  -----BEGIN CERTIFICATE-----
  MIIGDzCCA/egAwIBAgIUSN5qe5hK60T3jxllWcFtbIGG/9EwDQYJKoZIhvcNAQEL
  BQAwgZYxCzAJBgNVBAYTAlVTMRYwFAYDVQQIDA1Ob3J0aENhcm9saW5hMRAwDgYD
  VQQHDAdSYWxlaWdoMRAwDgYDVQQKDAdSZWQgSGF0MRIwEAYDVQQLDAlNYXJrZXRp
  bmcxNzA1BgNVBAMMLnByb3Zpc2lvbi5zY2htYXVzdGVjaC5zdHVkZW50cy5vc3Au
  b3BlbnRsYy5jb20wHhcNMjAwOTI4MTkyNTU5WhcNMjEwOTI4MTkyNTU5WjCBljEL
  MAkGA1UEBhMCVVMxFjAUBgNVBAgMDU5vcnRoQ2Fyb2xpbmExEDAOBgNVBAcMB1Jh
  bGVpZ2gxEDAOBgNVBAoMB1JlZCBIYXQxEjAQBgNVBAsMCU1hcmtldGluZzE3MDUG
  A1UEAwwucHJvdmlzaW9uLnNjaG1hdXN0ZWNoLnN0dWRlbnRzLm9zcC5vcGVudGxj
  LmNvbTCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCCAgoCggIBAL26kc+lUeU/W6PB
  sl2Hvd6bt7robNoNXsHasY8V2JCSAfhN7m98uTBm5DCcTvysAPEPsZfcbD/oMGhp
  IjKyJC339znbUycSDlEz+5Q88pqGXUcSoZMihOR5d5yFhoEmFKouK7BZy+yjweJg
  388JkBx+6Mj1cC2BRaxANRPj27b7nxg+OCaFRf7S1aoDsPEdhaBhpSQ3xta4u1dl
  InwQFjx0Wcw+Wgpiv/VaASdI02ssqM+dktaFwOlUy8JLefmJs12UMRg+M4NhAmDx
  2J9mkLkQTdbpW0+B32FyMqlE+R18pkOj0W43AqvBA5Jh8FHng7yn/LTCK1gCoyDP
  IYw3/LGBhAXEx1igCU6bcX8kiompemnbauD6hAfpFm7mKFfjiGQYoX/D+jmq/QvX
  JtuDGbRhIkqLwzLk+cxKUwr+4vQrt8knUwo5+0q37dkNdW2M9VM7RREKsl2L4atZ
  TFvZPjPUUjT1Y3X/5tJxKa0RULsjJkxmGF6l25hLeEolAeehGXFAuUU6PKkWEYRL
  wBkSQ7fX6drMBU4mTrkVZoX1fz8bsZVJQ2eLYWV7OKAPs7dMcZGkBacB7eHR+2x5
  BkYQrdmaT0YIttsg8g2tTj5DaUkSATdBPQ0tA7yTSE9AUsrRM3vVsgAfCylnwVEV
  bQEo/KYf67AYsolzMbinY2cwxCMjAgMBAAGjUzBRMB0GA1UdDgQWBBQW0tvxa/On
  OcERLPlJM/3cCQvkNDAfBgNVHSMEGDAWgBQW0tvxa/OnOcERLPlJM/3cCQvkNDAP
  BgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3DQEBCwUAA4ICAQCk0iM0Bq6nO34crslm
  bzgNYxLSaATpmKIlPMLtO6+tH1VXUeqGq1Ba0+x//jWIyUyLAztZRGiRA2zfZ/Gb
  IMC2iM1p/hGK2tvf0MY8yzU2rbwnS4Oelx8+of1LcHY28EmsaKetY8oo2LlSv2qF
  9+xzV7GMb/3+49xpFwd+d3cRGOS04LodxSjpPqUAVprXQQcU8vHINYg+Pe5zgSIM
  HvZCKXb78OPjmMRgRxdPdzeRRmfYGmiB+QdjSpir4V6aWO1Or/k1r9lqvE5xG8Xv
  /x6/e9QggMrcbzOfWfUNzw4aAKmb8c9qP2hz7BUirvLvKcpVq3RuhpBeGnZqsdYw
  f3xSKX5pks33WG9dlG9j58zSEU/R6Ck0Gh7IivHgwRdrRF/PO7EosZO20lmcf6X7
  q0klH1tvi79Si9VgZossvoHG0eO0KmPuNTmxOozMtNhFxZlrfsfZerpKX244tLhZ
  OJC4Uz5OXWZSZAe1jKvNS0lGrciX5MioHF9sELBdKTk9XotZLTGo/BVOl+QMstnJ
  PXOch5SceU6yDTmOWpSGlYN3oLSzPC0aO/wl5/qx+1vNoiD38A/i2DTm64bzWTzJ
  PRTMyM5MDADk4M1wbgSKNBWxrppMm1ThhlZfSjLFkyoSK3DxZ3Pp2jp4Qi9qmzYS
  L2Ylo/Zc//XX3sBLTR/MobiuVQ==
  -----END CERTIFICATE-----
~~~

You can see that cert is a standard cert like any other but it is required for our disconnected installation because it allows the installer to talk to the local registry via a TLS connection.



 
 
Show how the local registry is already configured 
Run through steps on how to pull down registry images to remote registry
Add bits to the install-config.yaml to reference local registry
Precache RHCOS images locally
Add bits to the install-config.yaml to reference local RHCOS image location
Add bits to the install-config.yaml to enable trust bundle for local registry
