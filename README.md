<img src="img/redhat.png" style="width: 350px;" border=0/>

# Deep-dive Hands-on with OpenShift Baremetal IPI & OpenShift Virtualization

## Lab Overview

Firstly, welcome to Red Hat Tech Ready 2020! The past few years have been an exciting time for both Red Hat and the OpenShift community; we've seen unprecedented interest and development in this new revolutionary technology and we're proud to be at the heart of it all. Red Hat is firmly committed to the future of OpenShift, Kubernetes, and the surrounding technologies; our goal is to continue to enhance the technology, make it more readily consumable and to enable our customers to be successful when using it.

One of the big steps forward was the introduction of tighter integration with the underlying infrastructure, allowing OpenShift to have more control over system deployment, maintenance, and configuration. This began with OpenShift 4.0 where we introduced tight integration for Amazon Web Services (AWS). For example, from a single bootstrap/provisioning node an administrator could bring up a new OpenShift cluster through direct interaction with AWS API's. This makes things like automatic scaling, deploying new nodes, and even upgrading cluster versions, easy and fully programmatic. 

As OpenShift continues to evolve with each new release we continue to add more integrations for a variety of infrastructure providers, such as OpenStack, VMware, and now, as per the purpose of this lab, bare metal. We're going to be walking you through the new baremetal flows that the OpenShift engineering organisation has been working on to bring the same capabilities  that our customers are used to with public cloud platforms to baremetal infrastructure through the use of the Kubernetes Native Infrastructure (KNI) concepts.

> **NOTE**: The features that we'll be using in this lab are [**technology preview**](https://access.redhat.com/support/offerings/techpreview) and aren't expected to GA until OpenShift 4.6, therefore there may be a few unexpected bugs/problems. Furthermore, due to the nature of the nested environment and providing things on-demand, you may run into some performance related timeouts; we've done our best to minimise these challenges and provided troubleshooting steps where possible. Please do reach out to the lab attendees if you get stuck!

## Lab Details

The aim of this lab is to get you, the attendees, a little bit more familiar with how an OpenShift Baremetal environment is built, what components it comprises of, and what it currently looks like. The rest of the materials will aim to do the following:

* Introduce the concepts of an OpenShift Baremetal IPI Install (with KNI), OpenShift Virtualization (previously known as CNV) and OpenShift Container Storage (OCS).

* Provide each student access to a dedicated environment which allows the lab to **simulate** an installation environment as close as feasibly possible to what a real Baremetal IPI setup might look like. Please note, this environment is virtualised to maximise our delivery footprint so will not be as performant as true bare metal.

* Allow a full end-to-end deployment of OpenShift Baremetal IPI with the enclosed instructions

* Demonstrate the deployment of some test workloads, based on both virtual machines and standard Kubernetes pods, backed by OpenShift Container Storage.

## Hands On!

We feel that getting hands on with Baremetal IPI is essential to helping you fully understand the details and specifics of this installtion method at a level even deeper than just slides alone. And of course, this training is designed to compliment and enhance the delivered messaging to better assist you in positioning, demonstrating, designing, and selling this important OpenShift installation pattern. 

If you have any problems with the lab or have any questions about Red Hat or our OpenShift offering, please let us know and a lab moderator will be available to assist. And don't worry, the online copy of the lab will be available for the foreseeable future for your review after the session.

And please do stay in touch as we very much welcome contributions and pull requests! You can also reach out to the Field PM team directly at field-engagement@redhat.com
