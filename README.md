# **Cluster Infrastructure Management Using PXE**

<img src="224_RHACM_Cluster_Lifecycle_Arch_0222.png" style="width: 1000px;" border=0/>

Anyone who has used Red Hat Advanced Cluster Management for Kubernetes knows it offers a multitude of ways to manage and deploy the OpenShift cluster lifecycle depending on whether one is deploying on prem or within one of the supported clouds.  In Red Hat Adavanced Cluster Management for Kubernetes 2.4 the Cluster Infrastructure Management interface, which is reliant on using the Assisted Installer behind the scenes, was introduced.  This option for deploying a baremetal cluster relied on server nodes that could support the mounting the discovery image via virtual media if the goal was to do zero touch provisioning.  However this lead to a gap for hardware that did not have the virtual media capability but still wanted to be deployed without having to actually take a USB stick with the image into the datacenter.

With the introduction of Red Hat Advanced Cluster Management for Kubernetes 2.5.1 we are introducting the capability of being able to PXE boot servers via the Cluster Infrastructure Management interface.  This feature will breath new life into those hosts that suffered the lack of remote virtual media mounts. Let's go ahead and see how this new option will work in a practical example.

First lets talk a bit about the reference architecture I am using.  I currently have a Red Hat Advanced Cluster Management for Kubernestes 2.5.1 hub cluster running on a 3 node compact OpenShift 4.10.16 cluster with OpenShift Data Foundation 4.10 acting as the storage for any of my persisten volume requirements.  Further I have a ISCP DHCP server setup on my private network of 192.168.0.0/24 to provide DHCP reservations for the 3 hosts we will discover via the PXE boot method and deploy OpenShift onto.

Before we can use the Cluster Infrastrutcture Management portion of Red Hat Advanced Cluster Management for Kubernetes we need to enable the metal3 provisioning configuration to watch all namespace.

~~~bash
$ oc patch provisioning provisioning-configuration --type merge -p '{"spec":{"watchAllNamespaces": true }}'
provisioning.metal3.io/provisioning-configuration patched
~~~

Next let's create the ClusterImageset resource yaml which will point to OpenShift 4.10.16:

~~~bash
$ cat << EOF > ~/kni20-clusterimageset.yaml
apiVersion: hive.openshift.io/v1
kind: ClusterImageSet
metadata:
  name: openshift-v4.10.16
  namespace: multicluster-engine
spec:
  releaseImage: quay.io/openshift-release-dev/ocp-release:4.10.16-x86_64
EOF  
~~~

Now let's apply the cluster imageset to the cluster:

~~~bash
$ oc create -f ~/kni20-clusterimageset.yaml 
clusterimageset.hive.openshift.io/openshift-v4.10.16 created
~~~

Next we need to create a mirror config which tells the assisted image service where to get the images from, since we are using a disconnected installation to deploy clustersN+1.  This configuration should contain the CA certificate and any registry references to ensure that the cluster being deployed knows to pull the images it needs from the local registry.

Let's create the config using the below:

~~~bash
$ cat << EOF > ~/kni20-mirror-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kni20-mirror-config
  namespace: multicluster-engine
  labels:
    app: assisted-service
data:
  registries.conf: |
    unqualified-search-registries = ["docker.io"]

    [[registry]]
      prefix = ""
      location = "registry.redhat.io/rhacm2"
      mirror-by-digest-only = true

    [[registry.mirror]]
      location = "quay.io:443/acm-d"

    [[registry]]
      prefix = ""
      location = "registry.redhat.io/multicluster-engine"
      mirror-by-digest-only = true

    [[registry.mirror]]
      location = "quay.io:443/acm-d"

    [[registry]]
      prefix = ""
      location = "registry.access.redhat.com/openshift4/ose-oauth-proxy"
      mirror-by-digest-only = true

    [[registry.mirror]]
      location = "registry.redhat.io/openshift4/ose-oauth-proxy"      
EOF
~~~

Now let's create the mirror configuration on cluster0:

~~~bash
$ oc create -f kni20-mirror-config.yaml 
configmap/kni20-mirror-config created
~~~

For Assisted Installer we need to create an agent service configuration resource that will tell the operator how much storage we need for the various components like database and filesystem. It will also define which OpenShift versions to maintain.

~~~bash
$ cat << EOF > ~/kni20-agentserviceconfig.yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: AgentServiceConfig
metadata:
 name: agent
spec:
  iPXEHTTPRoute: enabled
  databaseStorage:
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 20Gi
  filesystemStorage:
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 20Gi
  mirrorRegistryRef:
    name: "kni20-mirror-config"
  osImages:
    - openshiftVersion: "4.10"
      version: "410.84.202205191234-0"
      url: "https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/4.10/4.10.16/rhcos-4.10.16-x86_64-live.x86_64.iso"
      rootFSUrl: "https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/4.10/4.10.16/rhcos-4.10.16-x86_64-live-rootfs.x86_64.img"
      cpuArchitecture: "x86_64"
EOF
~~~

Once the agent service configuration file is created apply it to the cluster:

~~~bash
$ oc create -f ~/kni20-agentserviceconfig.yaml
agentserviceconfig.agent-install.openshift.io/agent created
~~~

After a few minutes validate that the pods for the Infrastructure operator have started:

~~~bash
$ oc get pods -n multicluster-engine |grep assisted
assisted-image-service-0                               0/1     Running   0          92s
assisted-service-5b65cfd866-sp2vs                      2/2     Running   0          93s


$ oc get pvc -n multicluster-engine
NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                  AGE
assisted-service   Bound    pvc-5d485331-c3e2-4331-bbe8-1635be66d07d   20Gi       RWO            ocs-storagecluster-ceph-rbd   114s
postgres           Bound    pvc-bcfa2e1c-63b7-4818-9848-00c6101efcce   20Gi       RWO            ocs-storagecluster-ceph-rbd   113s

$ oc get routes -n multicluster-engine
NAME                          HOST/PORT                                                               PATH   SERVICES                 PORT                          TERMINATION   WILDCARD
assisted-image-service        assisted-image-service-multicluster-engine.apps.kni20.schmaustech.com          assisted-image-service   assisted-image-service        reencrypt     None
assisted-image-service-ipxe   assisted-image-service-multicluster-engine.apps.kni20.schmaustech.com   /      assisted-image-service   assisted-image-service-http                 None
assisted-service              assisted-service-multicluster-engine.apps.kni20.schmaustech.com                assisted-service         assisted-service              reencrypt     None
assisted-service-ipxe         assisted-service-multicluster-engine.apps.kni20.schmaustech.com         /      assisted-service         assisted-service-http                       None
~~~

We have now completed the additional configuration steps needed for Cluster Infrastructure Management and Assisted Installer.  Now lets move onto defining the resources for spoke cluster which will be called kni21.  The first step here is to create a namespace which we will name after our cluster name:

~~~bash
$ oc create namespace kni21
namespace/kni21 created
~~~

Next we need to create a secret to store our pull-secret within that namespace:

~~~bash
$ oc create secret generic pull-secret -n kni21 --from-file=.dockerconfigjson=merged-pull-secret.json --type=kubernetes.io/dockerconfigjson
secret/pull-secret created
~~~

Next we will generate an infrastructure environment resource configuration that will contain an public SSH key:

~~~bash
$ cat << EOF > ~/kni21-infraenv.yaml
---
apiVersion: agent-install.openshift.io/v1beta1
kind: InfraEnv
metadata:
  name: kni21
  namespace: kni21
spec:
  agentLabels:
    project: kni21
  sshAuthorizedKey: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCoy2/8SC8K+9PDNOqeNady8xck4AgXqQkf0uusYfDJ8IS4pFh178AVkz2sz3GSbU41CMxO6IhyQS4Rga3Ft/VlW6ZAW7icz3mw6IrLRacAAeY1BlfxfupQL/yHjKSZRze9vDjfQ9UDqlHF/II779Kz5yRKYqXsCt+wYESU7DzdPuGgbEKXrwi9GrxuXqbRZOz5994dQW7bHRTwuRmF9KzU7gMtMCah+RskLzE46fc2e4zD1AKaQFaEm4aGbJjQkELfcekrE/VH3i35cBUDacGcUYmUEaco3c/+phkNP4Iblz4AiDcN/TpjlhbU3Mbx8ln6W4aaYIyC4EVMfgvkRVS1xzXcHexs1fox724J07M1nhy+YxvaOYorQLvXMGhcBc9Z2Au2GA5qAr5hr96AHgu3600qeji0nMM/0HoiEVbxNWfkj4kAegbItUEVBAWjjpkncbe5Ph9nF2DsBrrg4TsJIplYQ+lGewzLTm/cZ1DnIMZvTY/Vnimh7qa9aRrpMB0= bschmaus@provisioning"
  pullSecretRef:
    name: pull-secret
EOF
~~~

With the infrastructure environment file created lets apply it to the hub cluster:

~~~bash
$ oc create -f ~/kni21-infraenv.yaml
infraenv.agent-install.openshift.io/kni21 created
~~~

Now lets take a look at the infrastucture environment output.  Specifically we want to look the contents of the ipxeScript.  

~~~bash
$ oc get infraenv kni21 -n kni21 -o yaml|grep ipxeScript
    ipxeScript: http://assisted-service-multicluster-engine.apps.kni20.schmaustech.com/api/assisted-install/v2/infra-envs/a2ab51ad-c217-4bac-9a45-1d3eebb88b9a/downloads/files?api_key=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJpbmZyYV9lbnZfaWQiOiJhMmFiNTFhZC1jMjE3LTRiYWMtOWE0NS0xZDNlZWJiODhiOWEifQ.73dkqVX8IkrdvBBQln_KTDjzIdYR0YRv3Ky-rm8rxClqg-lz8PyrPdbkJXwFLusGwGDik6bBpGkx4Lf_cDeSWg&file_name=ipxe-script
~~~

~~~bash
curl -L -o ipxe -O 'http://assisted-service-multicluster-engine.apps.kni20.schmaustech.com/api/assisted-install/v2/infra-envs/a2ab51ad-c217-4bac9a45-1d3eebb88b9a/downloads/files?api_key=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJpbmZyYV9lbnZfaWQiOiJhMmFiNTFhZC1jMjE3LTRiYWMtOWE0NS0xZDNlZWJiODhiOWEifQ.73dkqVX8IkrdvBBQln_KTDjzIdYR0YRv3Ky-rm8rxClqg-lz8PyrPdbkJXwFLusGwGDik6bBpGkx4Lf_cDeSWg&file_name=ipxe-script'
~~~

~~~bash
$ cat ipxe
#!ipxe
initrd --name initrd http://assisted-image-service-multicluster-engine.apps.kni20.schmaustech.com/images/a2ab51ad-c217-4bac-9a45-1d3eebb88b9a/pxe-initrd?api_key=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJpbmZyYV9lbnZfaWQiOiJhMmFiNTFhZC1jMjE3LTRiYWMtOWE0NS0xZDNlZWJiODhiOWEifQ.9F_vEPli7TkAXAkFPWnFkeXK5xTMgVcwHScCuzi_DoWllbyoGgd9byl4W8yZOMSID2q_DvMSdY1ByLlMEqJcwQ&arch=x86_64&version=4.10
kernel http://assisted-image-service-multicluster-engine.apps.kni20.schmaustech.com/boot-artifacts/kernel?arch=x86_64&version=4.10 initrd=initrd coreos.live.rootfs_url=http://assisted-image-service-multicluster-engine.apps.kni20.schmaustech.com/boot-artifacts/rootfs?arch=x86_64&version=4.10 random.trust_cpu=on rd.luks.options=discard ignition.firstboot ignition.platform.id=metal console=tty1 console=ttyS1,115200n8 coreos.inst.persistent-kargs="console=tty1 console=ttyS1,115200n8"
boot
~~~

~~~
subnet 192.168.0.0 netmask 255.255.255.0 {
        option routers                  192.168.0.1;
        option subnet-mask              255.255.255.0;
        option domain-search            "schmaustech.com";
        option domain-name-servers      192.168.0.10;
        option time-offset              -18000;
        range   192.168.0.225   192.168.0.240;
        next-server 192.168.0.10;
        if exists user-class and option user-class = "iPXE" {
            filename "ipxe";
        } else {
            filename "lpxelinux.0";
        }
    }
}

host nuc1 {
   option host-name "nuc1.schmaustech.com";
   hardware ethernet b8:ae:ed:72:10:f6;
   fixed-address 192.168.0.2;
}

host nuc2 {
   option host-name "nuc2.schmaustech.com";
   hardware ethernet c0:3f:d5:6d:52:bc;
   fixed-address 192.168.0.3;
}

host nuc3 {
   option host-name "nuc3.schmaustech.com";
   hardware ethernet b8:ae:ed:71:fc:5f;
   fixed-address 192.168.0.6;
}
~~~


