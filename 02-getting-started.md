# Lab Instructions

These lab instructions are very self explanatory, in that we try our best to explain everything that's going on; give you the neccessary background into the tasks so you understand the concepts and why you're doing it, followed by screenshots and links where required, and crucially the commands you need to execute with expected outputs.

When we're asking you to execute terminal **commands** you'll see it in the following format:

~~~bash
user@the-system$ the-command
(the output)
~~~

For example:

~~~bash
rdo@laptop$ uptime
11:27  up 84 days, 16:48, 3 users, load averages: 1.21 1.40 1.52
~~~

This ensures that you're executing the correct commands on the correct system, with the correct user. The instructions are provided in an easy to copy & paste mechanism, although please don't use this lab as a copy & paste exercise; it's easy to make mistakes even when copying and it's not conducive to learning things properly :-).

> **NOTE**: If you see "**(...)**" in the command output, we've put this to advise that there's considerable output from the command being executed and it has been omitted to keep this lab guide clean.

# Lab Environment

Each attendee of this lab, regardless of whether you're following this after the Red Hat Tech Exchange has finished, will be utilising a public-cloud based dedicated environment, i.e. a fully virtualised environment that's just for you; you won't need to share and it doesn't matter if you break it - we can easily spin up some more environments, although you may run out of time during the allocated timeslots if you ask to use a different environment after you've started.

For the Red Hat Tech Exchange events we're utilising OpenTLC to provide the resources required for this lab, and whilst it may not be the most performant solution it's great for giving us on-demand, scalable, and flexible lab sessions. It's important to note that all of OpenTLC is virtualised, therefore all of the "baremetal" nodes you'll leverage are actually virtual machines, and we have to mimic IPMI (baremetal control) capabilities through the OpenTLC API, although all of this will have been pre-configured for you, so the process you follow in this lab will be no different to what you'd do with real baremetal. Ultimately, the aim of this lab is to make it "as close as feasibly possible" to a real-world scenario, albeit within the constraints of the Ravello platform.

Each dedicated environment has a '**provision**' host that acts as a gateway in, and is uniquely identified through DNS. Once you've connected to your provision virtual machine below, you'll be provided with instructions on how to complete the lab. Everyone's instructions are identical after connection to the provision host, as everyones environment has been provisioned from the same template.

The '**provision**' host has been prebuilt and will be used to bootstrap all of the OpenShift cluster on-top of the underlying "baremetal" infrastructure that will be automated through OpenStack Ironic and IPMI. This provisioner host has been partially configured for speed and convenience, but not in any way that will detract from the labs purpose, or cause confusion, this machine will be used extensively throughout the lab as our main point of execution.

There are many other dedicated machines (VM's in OpenTLC) that will make up the rest of the lab, some of the internals can be seen below, and will be accessible to you in your lab; and can be useful for viewing the console output of some of your systems, just be very careful with it when you're using it (a link will be displayed for you when you follow the instructions in the next section). For information purposes, here's mine:

<img src="img/hosts.png" style="width: 1000px;"/>

See below for a description of each of the hosts you can see:

| Node Name | Description |
|---|---|
| **provision** | The partially build-up provisioner host, a RHEL8 machine pre-configured with package access that will bootstrap the rest of the cluster as per the instructions in later lab sections.  This host is also our jumphost that we will connect directly to. |
| **master-0** | An empty, i.e. no operating system installed or configured, sytem that will become one of our OpenShift master systems. Note that this system has been configured on the RHPDS side to be connected to the correct networks.|
| **master-1** | As above, an additional master. |
| **master-2** | As above, an additional master. |
| **worker-0** | A further empty system that we'll use during the deployment of OpenShift cluster.  Further we will add additional storage for OCS during lab.|
| **worker-1** | As above, an additional worker. |
| **worker-2** | As above, an additiona worker that will be added after OpenShift cluster deployment |
| **bmc** | This host provides vBMC (i.e. virtual IPMI) capabilities and wraps IPMI commands back to the OpenStack API with an authentication token. This setup is automated and you don't have to worry about configuring it in this lab; assume that IPMI is automagically available in your lab already.<br /><br /> This machine also hosts DHCP and DNS for our environment, as will be required for all KNI deployments in the field, and also acts as a package repo server. |

# Connecting

You'll need to use your own laptop to connect into your (public cloud hosted) dedicated environment. It's on this environment that you'll perform the later lab instructions, and you should only need to use your terminal emulator and a web-browser to complete all of the tasks.

To get started, we need to request your own dedicated session, from the Content Hub for RHTR 2020 you will be directed to https://portal.opentlc.com/catalog/explorer#/ . What you should see is as follows, noting that if there's more than one lab shown in the drop-down box ensure you select **Hands-On w KNI for Baremetal OCP**:

> **NOTE:** The screenshot below was captured before this lab became generally available.  The lab will appear in the **RHTR 2020** catalog.

<img src="img/guid1.png" style="width: 1000px;"/>

This will allocate a pre-deployed session for your usage with a **GUID** that's used to uniquely identify your session. You will get an email on how to access the environment once it has been provisioned.  Here's an example below:

<img src="img/guid2.png" style="width: 1000px;"/>

The environment takes around 20 minutes to power-up, but don't be alarmed if you cannot connect in straight away, it may just require a few more minutes. Once it's up and running, we can connect to our jumphost:

~~~bash
[bschmaus@bschmaus ~]$ ssh lab-user@provision.hhnfk.dynamic.opentlc.com
The authenticity of host 'provision.hhnfk.dynamic.opentlc.com (150.239.28.71)' can't be established.
ECDSA key fingerprint is SHA256:wJtlUptPHgzqQWLzk0oRLA5sLkKR4hQ1NYbcF6gC4RQ.
ECDSA key fingerprint is MD5:03:14:41:53:46:6a:0c:8e:87:34:3a:65:58:91:17:20.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'provision.hhnfk.dynamic.opentlc.com,150.239.28.71' (ECDSA) to the list of known hosts.
lab-user@provision.hhnfk.dynamic.opentlc.com's password: 
Activate the web console with: systemctl enable --now cockpit.socket

This system is not registered to Red Hat Insights. See https://cloud.redhat.com/
To register this system, run: insights-client --register

[lab-user@provision ~]$ ls
pull-secret.json  scripts
[lab-user@provision ~]$ 
~~~

Now we're ready to proceed with the rest of our lab steps. If you had any problems getting access or if you have any questions, please feel free to ask any of the moderators at any time.
