# **PXE Artifacts in Red Hat Advanced Cluster Management for Kubernetes**

<img src="224_RHACM_Cluster_Lifecycle_Arch_0222.png" style="width: 1000px;" border=0/>

Anyone who has used Red Hat Advanced Cluster Management for Kubernetes knows it offers a multitude of ways to manage and deploy the OpenShift cluster lifecycle depending on whether one is deploying on prem or within one of the supported clouds.  In Red Hat Adavanced Cluster Management for Kubernetes 2.4 the Cluster Infrastructure Management interface, which is reliant on using the Assisted Installer behind the scenes, was introduced.  This option for deploying a baremetal cluster relied on server nodes that could support the mounting of the discovery image via virtual media if the goal was to do zero touch provisioning.  However this lead to a gap for hardware that did not have the virtual media capability but still wanted to be deployed without having to actually take a USB stick with the image into the datacenter.

With the introduction of Red Hat Advanced Cluster Management for Kubernetes 2.5.1 we now have the capability of being able to PXE boot servers via the Cluster Infrastructure Management interface.  This feature will breath new life into those hosts that suffered the lack of remote virtual media mounts. Let's go ahead and see how this new option will work in a practical example.

First lets talk a bit about the reference architecture I am using.  I currently have a Red Hat Advanced Cluster Management for Kubernestes 2.5.1 hub cluster running on a 3 node compact OpenShift 4.10.16 cluster with OpenShift Data Foundation 4.10 acting as the storage for any of my persistent volume requirements.  Further I have a DHCP server setup on my private network of 192.168.0.0/24 to provide DHCP reservations for the 3 hosts we will discover via the PXE boot method and deploy OpenShift onto.

Before we can use the Cluster Infrastrutcture Management portion of Red Hat Advanced Cluster Management for Kubernetes we need to enable the metal3 provisioning configuration to watch all namespaces.  We can do that with the following command if it has not been done already:

~~~bash
$ oc patch provisioning provisioning-configuration --type merge -p '{"spec":{"watchAllNamespaces": true }}'
provisioning.metal3.io/provisioning-configuration patched
~~~

Now lets get started by creating the ClusterImageset resource yaml which will point to OpenShift 4.10.16 release:

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

Next we will apply the cluster imageset to the hub cluster:

~~~bash
$ oc create -f ~/kni20-clusterimageset.yaml 
clusterimageset.hive.openshift.io/openshift-v4.10.16 created
~~~

Then we need to define a mirror config which tells the assisted image service where to get the images from, since I am using a prerelease version of Red Hat Advanced Cluster Management.  This step should not be needed when the release goes GA.  However it should be noted this step would be needed if performing a disconnected installation since we would need to tell the discovery ISO where to pull the images from if using a local registry or different registry other then the default. Let's create the configuration using the below example:

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

Now we will apply the mirror configuration on the hub cluster:

~~~bash
$ oc create -f kni20-mirror-config.yaml 
configmap/kni20-mirror-config created
~~~

For Assisted Installer we need to create an agent service configuration resource that will tell the operator how much storage we need for the various components like database and filesystem. It will also define which OpenShift versions to maintain.  However since we are going to use PXE as our discovery boot method we have also added in the iPXEHTTPRoute option set to enabled (by default it is disabled).  This will enable and expose HTTP routes that will serve up the required PXE artifacts. 

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

After a few minutes validate that the pods for the Assisted Installer have started and their corresponding PVCs exist:

~~~bash
$ oc get pods -n multicluster-engine |grep assisted
assisted-image-service-0                               0/1     Running   0          92s
assisted-service-5b65cfd866-sp2vs                      2/2     Running   0          93s

$ oc get pvc -n multicluster-engine
NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                  AGE
assisted-service   Bound    pvc-5d485331-c3e2-4331-bbe8-1635be66d07d   20Gi       RWO            ocs-storagecluster-ceph-rbd   114s
postgres           Bound    pvc-bcfa2e1c-63b7-4818-9848-00c6101efcce   20Gi       RWO            ocs-storagecluster-ceph-rbd   113s
~~~

Let's further validate that the HTTP routes I mentioned above that expose the PXE artifacts were also created.  We should see two http routes along with two others that are encrypted.

~~~bash
$ oc get routes -n multicluster-engine
NAME                          HOST/PORT                                                               PATH   SERVICES                 PORT                          TERMINATION   WILDCARD
assisted-image-service        assisted-image-service-multicluster-engine.apps.kni20.schmaustech.com          assisted-image-service   assisted-image-service        reencrypt     None
assisted-image-service-ipxe   assisted-image-service-multicluster-engine.apps.kni20.schmaustech.com   /      assisted-image-service   assisted-image-service-http                 None
assisted-service              assisted-service-multicluster-engine.apps.kni20.schmaustech.com                assisted-service         assisted-service              reencrypt     None
assisted-service-ipxe         assisted-service-multicluster-engine.apps.kni20.schmaustech.com         /      assisted-service         assisted-service-http                       None
~~~

We have now completed the additional configuration steps needed for Cluster Infrastructure Management and Assisted Installer to serve up the PXE artifacts.  Now lets move onto defining the resources for spoke cluster which will be called kni21.  The first step here is to create a namespace which we will name after our cluster name:

~~~bash
$ oc create namespace kni21
namespace/kni21 created
~~~

In the namespace we just created lets create a secret to store our pull-secret:

~~~bash
$ oc create secret generic pull-secret -n kni21 --from-file=.dockerconfigjson=merged-pull-secret.json --type=kubernetes.io/dockerconfigjson
secret/pull-secret created
~~~

Next we will generate an infrastructure environment resource configuration that will contain an public SSH key and reference the pull-secret secret we created previoulsy in this namespace:

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

We can curl the script down so we can examine its contents to get a better understanding of what it does:

~~~bash
curl -L -o ipxe -O 'http://assisted-service-multicluster-engine.apps.kni20.schmaustech.com/api/assisted-install/v2/infra-envs/a2ab51ad-c217-4bac9a45-1d3eebb88b9a/downloads/files?api_key=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJpbmZyYV9lbnZfaWQiOiJhMmFiNTFhZC1jMjE3LTRiYWMtOWE0NS0xZDNlZWJiODhiOWEifQ.73dkqVX8IkrdvBBQln_KTDjzIdYR0YRv3Ky-rm8rxClqg-lz8PyrPdbkJXwFLusGwGDik6bBpGkx4Lf_cDeSWg&file_name=ipxe-script'
~~~

With the contents downloaded lets cat out the file which is basically a iPXE script.  The script basically contains the details of where to pull the initial boot kernel, initrd and rootfs.  These are all the components that make up the discovery ISO only we have broken them out as artifacts for the sake of PXE booting.

~~~bash
$ cat ipxe
#!ipxe
initrd --name initrd http://assisted-image-service-multicluster-engine.apps.kni20.schmaustech.com/images/a2ab51ad-c217-4bac-9a45-1d3eebb88b9a/pxe-initrd?api_key=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJpbmZyYV9lbnZfaWQiOiJhMmFiNTFhZC1jMjE3LTRiYWMtOWE0NS0xZDNlZWJiODhiOWEifQ.9F_vEPli7TkAXAkFPWnFkeXK5xTMgVcwHScCuzi_DoWllbyoGgd9byl4W8yZOMSID2q_DvMSdY1ByLlMEqJcwQ&arch=x86_64&version=4.10
kernel http://assisted-image-service-multicluster-engine.apps.kni20.schmaustech.com/boot-artifacts/kernel?arch=x86_64&version=4.10 initrd=initrd coreos.live.rootfs_url=http://assisted-image-service-multicluster-engine.apps.kni20.schmaustech.com/boot-artifacts/rootfs?arch=x86_64&version=4.10 random.trust_cpu=on rd.luks.options=discard ignition.firstboot ignition.platform.id=metal console=tty1 console=ttyS1,115200n8 coreos.inst.persistent-kargs="console=tty1 console=ttyS1,115200n8"
boot
~~~

To consume the iPXE script we can take a couple of different approaches.  The most hands off way would be to call the iPXE script directly but that also assumes the system booting has iPXE in the network interface firmware.   In my example I chose to host the ipxe script myself and chain any PXE clients to boot into an iPXE kernel to pick it up.   The snippets from my relevant DHCP config are below along with the host reservations:

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
            filename "pxelinux.0";
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

Along with the DHCP configuration I have the following files present in my tftboot location.  What happens in my DHCP chaining example is that a PXE client will send out a boot request and DHCP will see its a PXE client and serve it pxelinux.0 which points to a default config under the pxelinux.cfg directory.  The default config is just a pointer to an iPXE kernel which will then be called and request DHCP again which now is seen as an iPXE client and will execute the iPXE script to start the boot process.

~~~bash
$ pwd
/var/lib/tftpboot
$ ls
ipxe  ipxe.efi  ipxe.lkrn  ldlinux.c32  libcom32.c32  libutil.c32  menu.c32  pxelinux.0  pxelinux.cfg  undionly.kpxe  vesamenu.c32
$ cat ./pxelinux.cfg/default 
DEFAULT ipxe.lkrn
~~~

With our DHCP configurations in place we can now boot our 3 nodes that will be used to deploy our kni21 cluster.  The following screenshot is an example of what one would expect to see during the PXE chaining process.

<img src="ipxe-dhcp.jpg" style="width: 1000px;" border=0/>

After booting the nodes and waiting for a few minutes we should now see agents reporting in when looking at the agent resource in the kni21 namespace:

~~~bash
$ oc get agent -n kni21 
NAME                                   CLUSTER   APPROVED   ROLE          STAGE
07f25812-6c5b-ece8-a4d5-9a3c2c76fa3a             false      auto-assign   
cef2bbd6-f974-5ecf-331e-db11391fd7a5             false      auto-assign   
d1e0c4b8-6f70-8d5b-93a3-706754ee2ee9             false      auto-assign 
~~~

With the hosts booted and showing under the agent we now need to create the spoke cluster manifests which will define what our cluster kni21 should look like.  There is some information we need to feed the manifest to define the configuration of the spoke cluster:

AgentClusterInstall will hold the configuration for the spoke cluster that we're about to provision.  It will also be the resource that we can watch during the deployment process in the event we need to debug issues.

We need to tweak it accordingly:
- `imageSetRef` - This is the ClusterImageSet that will be used (Openshift Version)
- `apiVIP` and `ingressVIP` - The IPs we reserved for API and Ingress usage
- `clusterNetwork`.cidr - CIDR for the kubernetes pods
- `clusterNetwork`.hostPrefix - Network prefix that will determine how many IPs are reserved for each node
- `serviceNetwork` - The CIDR that will be used for kubernetes services
- `controlPlaneAgents` - The number of control plane nodes we will be provisioning
- `workerAgents` - Number of worker agents we are provisioning now
- `sshPublicKey` - ssh public key that will be added to `core` username's authorized_keys in every node

~~~bash
$ cat << EOF > ~/cluster-kni21.yaml
---
apiVersion: extensions.hive.openshift.io/v1beta1
kind: AgentClusterInstall
metadata:
  name: kni21
  namespace: kni21
spec:
  clusterDeploymentRef:
    name: kni21
  imageSetRef:
    name: openshift-v4.10.16
  apiVIP: "192.168.0.120"
  ingressVIP: "192.168.0.121"
  networking:
    clusterNetwork:
      - cidr: "10.128.0.0/14"
        hostPrefix: 23
    serviceNetwork:
      - "172.30.0.0/16"
  provisionRequirements:
    controlPlaneAgents: 3
    workerAgents: 0
  sshPublicKey: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCoy2/8SC8K+9PDNOqeNady8xck4AgXqQkf0uusYfDJ8IS4pFh178AVkz2sz3GSbU41CMxO6IhyQS4Rga3Ft/VlW6ZAW7icz3mw6IrLRacAAeY1BlfxfupQL/yHjKSZRze9vDjfQ9UDqlHF/II779Kz5yRKYqXsCt+wYESU7DzdPuGgbEKXrwi9GrxuXqbRZOz5994dQW7bHRTwuRmF9KzU7gMtMCah+RskLzE46fc2e4zD1AKaQFaEm4aGbJjQkELfcekrE/VH3i35cBUDacGcUYmUEaco3c/+phkNP4Iblz4AiDcN/TpjlhbU3Mbx8ln6W4aaYIyC4EVMfgvkRVS1xzXcHexs1fox724J07M1nhy+YxvaOYorQLvXMGhcBc9Z2Au2GA5qAr5hr96AHgu3600qeji0nMM/0HoiEVbxNWfkj4kAegbItUEVBAWjjpkncbe5Ph9nF2DsBrrg4TsJIplYQ+lGewzLTm/cZ1DnIMZvTY/Vnimh7qa9aRrpMB0= bschmaus@provisioning"
EOF
~~~

In ClusterDeployment resource, which we are appending to the already created cluster-kni21.yaml, we need define:
- `baseDomain`
- `clusterName`
- The name of the `AgentClusterInstall` resource associated to this cluster
- An `agentSelector` that matches the agents labels
- The `Secret` containing our pullSecret

~~~bash
$ cat << EOF >> ~/cluster-kni21.yaml
---
apiVersion: hive.openshift.io/v1
kind: ClusterDeployment
metadata:
  name: kni21
  namespace: kni21
spec:
  baseDomain: schmaustech.com
  clusterName: kni21
  controlPlaneConfig:
    servingCertificates: {}
  installed: false
  clusterInstallRef:
    group: extensions.hive.openshift.io
    kind: AgentClusterInstall
    name: kni21
    version: v1beta1
  platform:
    agentBareMetal:
      agentSelector:
        matchLabels:
          project: kni21
  pullSecretRef:
    name: pull-secret
EOF
~~~

The KlusterletAddonConfig and ManagedCluster components which will also be appended do not need much tweaking:

~~~bash
$ cat << EOF >> ~/cluster-kni21.yaml
---
apiVersion: agent.open-cluster-management.io/v1
kind: KlusterletAddonConfig
metadata:
  name: kni21
  namespace: kni21
spec:
  clusterName: kni21
  clusterNamespace: kni21
  clusterLabels:
    cloud: auto-detect
    vendor: auto-detect
  applicationManager:
    enabled: false
  certPolicyController:
    enabled: false
  iamPolicyController:
    enabled: false
  policyController:
    enabled: false
  searchCollector:
    enabled: false
    
---
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: kni21
  namespace: kni21
spec:
  hubAcceptsClient: true
EOF
~~~

With the resource file cluster-kni21.yaml created lets go ahead and apply it to our hub cluster:

~~~bash
$ oc create -f ~/cluster-kni21.yaml
agentclusterinstall.extensions.hive.openshift.io/cluster1 created
clusterdeployment.hive.openshift.io/cluster1 created
klusterletaddonconfig.agent.open-cluster-management.io/cluster1 created
managedcluster.cluster.open-cluster-management.io/cluster1 created
~~~

Now that we have our kni21 spoke cluster defined lets go ahead and associate the nodes we discovered with our agent to the cluster.  We do this by binding them to the cluster we just defined:

~~~bash
$ oc get agent -n kni21 -o json | jq -r '.items[] | select(.spec.approved==false) | .metadata.name' | xargs oc -n kni21 patch -p '{"spec":{"clusterDeploymentName":{"name": "kni21", "namespace": "kni21"}}}' --type merge agent
agent.agent-install.openshift.io/07f25812-6c5b-ece8-a4d5-9a3c2c76fa3a patched
agent.agent-install.openshift.io/cef2bbd6-f974-5ecf-331e-db11391fd7a5 patched
agent.agent-install.openshift.io/d1e0c4b8-6f70-8d5b-93a3-706754ee2ee9 patched
~~~

After binding the agents the nodes should look like this:

~~~bash
$  oc get agent -n kni21
NAME                                   CLUSTER   APPROVED   ROLE          STAGE
07f25812-6c5b-ece8-a4d5-9a3c2c76fa3a   kni21     false      auto-assign   
cef2bbd6-f974-5ecf-331e-db11391fd7a5   kni21     false      auto-assign   
d1e0c4b8-6f70-8d5b-93a3-706754ee2ee9   kni21     false      auto-assign   
~~~

Then we can either manually approve each agent with the following:

~~~bash
$ oc -n kni21 patch -p '{"spec":{"approved":true}}' --type merge agent <AGENT_ID_NAME>
~~~

Or we can approve them all as we did when we bound them in a previous step.  Note that the cluster installation will not proceed until they are approved.

~~~bash
$ oc get agent -n kni21 -ojson | jq -r '.items[] | select(.spec.approved==false) | .metadata.name'| xargs oc -n kni21 patch -p '{"spec":{"approved":true}}' --type merge agent
agent.agent-install.openshift.io/07f25812-6c5b-ece8-a4d5-9a3c2c76fa3a patched
agent.agent-install.openshift.io/cef2bbd6-f974-5ecf-331e-db11391fd7a5 patched
agent.agent-install.openshift.io/d1e0c4b8-6f70-8d5b-93a3-706754ee2ee9 patched

$ oc get agent -n kni21
NAME                                   CLUSTER   APPROVED   ROLE     STAGE
07f25812-6c5b-ece8-a4d5-9a3c2c76fa3a   kni21     true       master   
cef2bbd6-f974-5ecf-331e-db11391fd7a5   kni21     true       master   
d1e0c4b8-6f70-8d5b-93a3-706754ee2ee9   kni21     true       master  
~~~

Now all we need is to wait for the cluster to be fully provisioned.  There are a few commands we can run while its deploying if one wants to check on the deployment process.  Here are a few suggestions with the associated output:

~~~bash
$ oc get AgentClusterInstall -n kni21
NAME    CLUSTER   STATE
kni21   kni21     installing

$ oc get AgentClusterInstall -n kni21 -o yaml
apiVersion: v1
items:
- apiVersion: extensions.hive.openshift.io/v1beta1
  kind: AgentClusterInstall
  metadata:
    creationTimestamp: "2022-07-01T16:21:01Z"
    finalizers:
    - agentclusterinstall.agent-install.openshift.io/ai-deprovision
    generation: 3
    name: kni21
    namespace: kni21
    ownerReferences:
    - apiVersion: hive.openshift.io/v1
      kind: ClusterDeployment
      name: kni21
      uid: 733dbd01-c811-4553-9563-d98c49902dd0
    resourceVersion: "3000112"
    uid: 3eb51c0b-1fa9-4eec-8527-fc1b1aff4e79
  spec:
    apiVIP: 192.168.0.120
    clusterDeploymentRef:
      name: kni21
    clusterMetadata:
      adminKubeconfigSecretRef:
        name: kni21-admin-kubeconfig
      clusterID: 1881bae0-22a2-4e18-a921-aee98113dfde
      infraID: 37ae4a10-7491-4ba1-8559-6d504519c765
    imageSetRef:
      name: openshift-v4.10.16
    ingressVIP: 192.168.0.121
    networking:
      clusterNetwork:
      - cidr: 10.128.0.0/14
        hostPrefix: 23
      serviceNetwork:
      - 172.30.0.0/16
    provisionRequirements:
      controlPlaneAgents: 3
    sshPublicKey: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCoy2/8SC8K+9PDNOqeNady8xck4AgXqQkf0uusYfDJ8IS4pFh178AVkz2sz3GSbU41CMxO6IhyQS4Rga3Ft/VlW6ZAW7icz3mw6IrLRacAAeY1BlfxfupQL/yHjKSZRze9vDjfQ9UDqlHF/II779Kz5yRKYqXsCt+wYESU7DzdPuGgbEKXrwi9GrxuXqbRZOz5994dQW7bHRTwuRmF9KzU7gMtMCah+RskLzE46fc2e4zD1AKaQFaEm4aGbJjQkELfcekrE/VH3i35cBUDacGcUYmUEaco3c/+phkNP4Iblz4AiDcN/TpjlhbU3Mbx8ln6W4aaYIyC4EVMfgvkRVS1xzXcHexs1fox724J07M1nhy+YxvaOYorQLvXMGhcBc9Z2Au2GA5qAr5hr96AHgu3600qeji0nMM/0HoiEVbxNWfkj4kAegbItUEVBAWjjpkncbe5Ph9nF2DsBrrg4TsJIplYQ+lGewzLTm/cZ1DnIMZvTY/Vnimh7qa9aRrpMB0=
      bschmaus@provisioning
  status:
    apiVIP: 192.168.0.120
    conditions:
    - lastProbeTime: "2022-07-01T16:21:04Z"
      lastTransitionTime: "2022-07-01T16:21:04Z"
      message: SyncOK
      reason: SyncOK
      status: "True"
      type: SpecSynced
    - lastProbeTime: "2022-07-01T16:23:32Z"
      lastTransitionTime: "2022-07-01T16:23:32Z"
      message: The cluster's validations are passing
      reason: ValidationsPassing
      status: "True"
      type: Validated
    - lastProbeTime: "2022-07-01T16:23:32Z"
      lastTransitionTime: "2022-07-01T16:23:32Z"
      message: The cluster requirements are met
      reason: ClusterAlreadyInstalling
      status: "True"
      type: RequirementsMet
    - lastProbeTime: "2022-07-01T16:26:02Z"
      lastTransitionTime: "2022-07-01T16:26:02Z"
      message: 'The installation is in progress: Installation in progress'
      reason: InstallationInProgress
      status: "False"
      type: Completed
    - lastProbeTime: "2022-07-01T16:21:04Z"
      lastTransitionTime: "2022-07-01T16:21:04Z"
      message: The installation has not failed
      reason: InstallationNotFailed
      status: "False"
      type: Failed
    - lastProbeTime: "2022-07-01T16:21:04Z"
      lastTransitionTime: "2022-07-01T16:21:04Z"
      message: The installation is waiting to start or in progress
      reason: InstallationNotStopped
      status: "False"
      type: Stopped
    connectivityMajorityGroups: '{"192.168.0.0/24":["07f25812-6c5b-ece8-a4d5-9a3c2c76fa3a","cef2bbd6-f974-5ecf-331e-db11391fd7a5","d1e0c4b8-6f70-8d5b-93a3-706754ee2ee9"],"IPv4":["07f25812-6c5b-ece8-a4d5-9a3c2c76fa3a","cef2bbd6-f974-5ecf-331e-db11391fd7a5","d1e0c4b8-6f70-8d5b-93a3-706754ee2ee9"],"IPv6":["07f25812-6c5b-ece8-a4d5-9a3c2c76fa3a","cef2bbd6-f974-5ecf-331e-db11391fd7a5","d1e0c4b8-6f70-8d5b-93a3-706754ee2ee9"]}'
    debugInfo:
      eventsURL: https://assisted-service-multicluster-engine.apps.kni20.schmaustech.com/api/assisted-install/v2/events?api_key=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJjbHVzdGVyX2lkIjoiMzdhZTRhMTAtNzQ5MS00YmExLTg1NTktNmQ1MDQ1MTljNzY1In0.p1LuadM3n_NbbjgUP6MvL-34wifAy2aVqgL9K1b54bIrUbN4QyK2Kdtxt03JJxsH4rL-PNnenGkrTLEfhyvLnw&cluster_id=37ae4a10-7491-4ba1-8559-6d504519c765
      logsURL: ""
      state: installing
      stateInfo: Installation in progress
    ingressVIP: 192.168.0.121
    machineNetwork:
    - cidr: 192.168.0.0/24
    progress:
      totalPercentage: 50
    validationsInfo:
      configuration:
      - id: pull-secret-set
        message: The pull secret is set.
        status: success
      hosts-data:
      - id: all-hosts-are-ready-to-install
        message: All hosts in the cluster are ready to install.
        status: success
      - id: sufficient-masters-count
        message: The cluster has a sufficient number of master candidates.
        status: success
      network:
      - id: api-vip-defined
        message: The API virtual IP is defined.
        status: success
      - id: api-vip-valid
        message: api vip 192.168.0.120 belongs to the Machine CIDR and is not in use.
        status: success
      - id: cluster-cidr-defined
        message: The Cluster Network CIDR is defined.
        status: success
      - id: dns-domain-defined
        message: The base domain is defined.
        status: success
      - id: ingress-vip-defined
        message: The Ingress virtual IP is defined.
        status: success
      - id: ingress-vip-valid
        message: ingress vip 192.168.0.121 belongs to the Machine CIDR and is not
          in use.
        status: success
      - id: machine-cidr-defined
        message: The Machine Network CIDR is defined.
        status: success
      - id: machine-cidr-equals-to-calculated-cidr
        message: The Cluster Machine CIDR is equivalent to the calculated CIDR.
        status: success
      - id: network-prefix-valid
        message: The Cluster Network prefix is valid.
        status: success
      - id: network-type-valid
        message: The cluster has a valid network type
        status: success
      - id: networks-same-address-families
        message: Same address families for all networks.
        status: success
      - id: no-cidrs-overlapping
        message: No CIDRS are overlapping.
        status: success
      - id: ntp-server-configured
        message: No ntp problems found
        status: success
      - id: service-cidr-defined
        message: The Service Network CIDR is defined.
        status: success
      operators:
      - id: cnv-requirements-satisfied
        message: cnv is disabled
        status: success
      - id: lso-requirements-satisfied
        message: lso is disabled
        status: success
      - id: odf-requirements-satisfied
        message: odf is disabled
        status: success
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
~~~

Once the cluster installation completes in my case after around 60 minutes we can confirm the spoke cluster installation is done by executing the following command to see the agent status: 

~~~bash
$ oc get agent -n kni21 -w
NAME                                   CLUSTER   APPROVED   ROLE     STAGE
07f25812-6c5b-ece8-a4d5-9a3c2c76fa3a   kni21     true       master   Done
cef2bbd6-f974-5ecf-331e-db11391fd7a5   kni21     true       master   Done
d1e0c4b8-6f70-8d5b-93a3-706754ee2ee9   kni21     true       master   Done
~~~

We can also confirm the spoke cluster is available by checking the managedcluster list on the hub cluster:

~~~bash
$ oc get managedcluster
NAME            HUB ACCEPTED   MANAGED CLUSTER URLS                     JOINED   AVAILABLE   AGE
kni21           true           https://api.kni21.schmaustech.com:6443   True     True        80m
local-cluster   true           https://api.kni20.schmaustech.com:6443   True     True        21h
~~~

At this point we can extract the kubeconfig file from its secret in the hub cluster's namespace and run some validation checking to ensure our kni21 spoke cluster is up and running:

~~~bash
$ oc get secret -n kni21 kni21-admin-kubeconfig  -ojsonpath='{.data.kubeconfig}'| base64 -d > cluster-kni21-kubeconfig
~~~

And now we can query the spoke cluster API with the following commands:

~~~bash
$ KUBECONFIG=cluster-kni21-kubeconfig oc get node
NAME                   STATUS   ROLES           AGE   VERSION
nuc1.schmaustech.com   Ready    master,worker   69m   v1.23.5+3afdacb
nuc2.schmaustech.com   Ready    master,worker   68m   v1.23.5+3afdacb
nuc3.schmaustech.com   Ready    master,worker   48m   v1.23.5+3afdacb

$ KUBECONFIG=cluster-kni21-kubeconfig oc get clusteroperators
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.10.16   True        False         False      34m     
baremetal                                  4.10.16   True        False         False      64m     
cloud-controller-manager                   4.10.16   True        False         False      68m     
cloud-credential                           4.10.16   True        False         False      74m     
cluster-autoscaler                         4.10.16   True        False         False      64m     
config-operator                            4.10.16   True        False         False      66m     
console                                    4.10.16   True        False         False      42m     
csi-snapshot-controller                    4.10.16   True        False         False      65m     
dns                                        4.10.16   True        False         False      64m     
etcd                                       4.10.16   True        False         False      63m     
image-registry                             4.10.16   True        False         False      53m     
ingress                                    4.10.16   True        False         False      55m     
insights                                   4.10.16   True        False         False      59m     
kube-apiserver                             4.10.16   True        False         False      53m     
kube-controller-manager                    4.10.16   True        False         False      60m     
kube-scheduler                             4.10.16   True        False         False      59m     
kube-storage-version-migrator              4.10.16   True        False         False      66m     
machine-api                                4.10.16   True        False         False      56m     
machine-approver                           4.10.16   True        False         False      64m     
machine-config                             4.10.16   True        False         False      45m     
marketplace                                4.10.16   True        False         False      64m     
monitoring                                 4.10.16   True        False         False      49m     
network                                    4.10.16   True        False         False      66m     
node-tuning                                4.10.16   True        False         False      64m     
openshift-apiserver                        4.10.16   True        False         False      45m     
openshift-controller-manager               4.10.16   True        False         False      61m     
openshift-samples                          4.10.16   True        False         False      55m     
operator-lifecycle-manager                 4.10.16   True        False         False      65m     
operator-lifecycle-manager-catalog         4.10.16   True        False         False      65m     
operator-lifecycle-manager-packageserver   4.10.16   True        False         False      56m     
service-ca                                 4.10.16   True        False         False      66m     
storage                                    4.10.16   True        False         False      67m 

$ KUBECONFIG=cluster-kni21-kubeconfig oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.10.16   True        False         34m     Cluster version is 4.10.16
~~~

Everything looks good from the output above and to imagine it was all deployed using the PXE artifacts that are now available in Red Hat Advanced Cluster Management 2.5.1.  
