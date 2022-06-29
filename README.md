# **Cluster Infrastructure Management Using PXE**

<img src="image.jpg" style="width: 800px;" border=0/>

Anyone who has used Red Hat Advanced Cluster Management for Kubernetes knows it offers a multitude of ways to manage and deploy the OpenShift cluster lifecycle depending on whether one is deploying on prem or within one of the supported clouds.  In Red Hat Adavanced Cluster Management for Kubernetes 2.4 the Cluster Infrastructure Management interface, which is reliant on using the Assisted Installer behind the scenes, was introduced.  This option for deploying a baremetal cluster relied on server nodes that could support the mounting the discovery image via virtual media if the goal was to do zero touch provisioning.  However this lead to a gap for hardware that did not have the virtual media capability but still wanted to be deployed without having to actually take a USB stick with the image into the datacenter.

With the introduction of Red Hat Advanced Cluster Management for Kubernetes 2.5.1 we are introducting the capability of being able to PXE boot servers via the Cluster Infrastructure Management interface.  This feature will breath new life into those hosts that suffered the lack of remote virtual media mounts. Let's go ahead and see how this new option will work in a practical example.

First lets talk a bit about the reference architecture I am using.  I currently have a Red Hat Advanced Cluster Management for Kubernestes 2.5.1 hub cluster running on a 3 node compact OpenShift 4.10.16 cluster with OpenShift Data Foundation 4.10 acting as the storage for any of my persisten volume requirements.  Further I have a ISCP DHCP server setup on my private network of 192.168.0.0/24 to provide DHCP reservations for the 3 hosts we will discover via the PXE boot method and deploy OpenShift onto.



~~~bash

~~~
