<img src="img/redhat.png" style="width: 350px;" border=0/>

<h2>Deep-dive Hands-on with OpenShift Baremetal IPI & OpenShift Virtualization</h2>

#**Lab Overview**

Firstly, welcome to Red Hat Tech Exchange 2020! The past few years have been an exciting time for both Red Hat and the OpenShift community; we've seen unprecedented interest and development in this new revolutionary technology and we're proud to be at the heart of it all. Red Hat is firmly committed to the future of OpenShift, Kubernetes, and the surrounding technologies; our goal is to continue to enhance the technology, make it more readily consumable and to enable our customers to be successful when using it.

One of the big steps forward for us was to introduce tighter integration with the underlying infrastructure, allowing OpenShift to have more control over system deployment, maintenance, and configuration. With OpenShift 4.0 we came out with tight integration for Amazon Web Services, i.e. from a single bootstrap/provisioning node it could bring up a new OpenShift cluster that had control over AWS directly so it could automatically scale, build new nodes when required, upgrade cluster versions, and so on. As OpenShift 4.x continues to evolve we're adding further integrations such as OpenStack, VMware, and now, as per the purpose of this lab, bare metal.

This lab is primarily targeted to get you, the attendees, a little bit more familiar with how Kubernetes-native Infrastructure (KNI) is built, what components it comprises of, and what it currently looks like in its early-development stage. The rest of the materials will aim to do the following-

* Introduce the concepts of OpenShift Baremetal IPI Install, OpenShift virtualization and OpenShift Container Storage.

* Provide access to a dedicated environment that will allow the installation to simulate an environment that's as close as feasibly possible to what Baremetal IPI will look like, albeit virtualised and not totally representative.

* Allow a full, end-to-end deployment of OpenShift Baremetal IPI with the current instructions.

* Demonstrate the deployment of some test workloads based on both virtual machines and standard Kubernetes pods, backed by OpenShift Container Storage (Rook/OCS).

We feel that giving you a hands-on lab of Baremetal IPI will be a lot more beneficial than just delivering slideware. If you have any problems at all or have any questions about Red Hat or our OpenShift offering, please put your hand-up and a lab moderator will be with you shortly to assist - we've asked some of our OpenShift/KNI experts to be here today, so please make use of their time. If you have printed materials, they're yours to take away with you, otherwise this online copy will be available for the foreseeable future; I hope that they'll be useful assets in the future.
