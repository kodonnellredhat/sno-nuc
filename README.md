# Single Node OpenShift Disconnected on Intel Nuc
![image of networking with nuc](images/network-internet-connected.excalidraw.svg)

In this setup we have the below hardware, take note that the GL wifi router is only connected to the internet during the OCP Bin pull (collect_ocp) and the registry_mirror OCP hydration process

For DNS we will use the GL.iNet wifi router, and for the subnet we will use the default 192.168.8.0/24

* Thinkpad X1 Nano
  * 200+ gb free space
* Intel Nuc 13 Pro Kit
  * 64 gb DDR4
  * Samsung 2TB nvme
* GL.iNet GL-AXT1800
* Sandisk usbc USB 1.2 gb

## Basic Hardware Setup

### GL-AXT1800
* Standard setup, default wifi pwd on sticker as "Key"
  * Cable LAN2 to your internet uplink or setup wifi uplink to your home router via https://192.168.8.1/#/internet
### ThinkPad X1 Nano
* Fedora41 standard install
  * wifi connected to GL-ATX1800
  * sudo dnf install podman git nmstatectl -y
  * sudo dnf update -y
  * hostnamectl laptop.kmod.io
### Intel Nuc 13 Pro Kit
* Standard bios setup
  * hard wired to gbe LAN1(GL-ATX1800)
  * Current ## issue, bios time needs to be set to local time zone time

## DNS on GL-AXT1800 - OCP Requirements
https://192.168.8.1/cgi-bin/luci/admin/network/dhcp

| Item | Value |
| ------------ | ----------- |
| cluster name | ocp.kmod.io |
| api          | 192.168.8.199 |
| master0      | 192.168.8.199 |
| laptop       | 192.168.8.134 |
| apps         | 192.168.8.199 |

*note:* rendezvousIP will be 192.168.8.199 in this case

![dns setup](images/dns.png)

*Current hardware setup*
----------------------------------------------

![photo of current setup](images/hardware-setup.png)

## Laptop Prep Work 

This will collect the reqired OpenShift tools to executed a SNO (Single Node OpenShift) install on the Intel Nuc (Disconnected). As well as get the repo and images required for the disconnected install. 

### Clone a base repo to collect the ocp binary, the collect_ocp will download the below files

- oc-mirror
- openshift-install
- openshift-client (OC, kubectl)
- mirror-registry
- butane

> cd ~ && git clone https://github.com/kodonnellredhat/ocp417.git && cd ocp417
>
> ./collect_ocp

### Setup a registry on the laptop (Quay Light (Mirror Registry))

> mirror-registry/mirror-registry install

Save your output

![mirror-registry-output](images/mirror-registry-output.png)

### Configure the firewall to allow inbound access to the registry

>sudo firewall-cmd --permanent --add-port=80/tcp 
>
>sudo firewall-cmd --permanent --add-port=443/tcp 
>
>sudo firewall-cmd --permanent --add-port=8443/tcp

### Copy the mirror registry Cert into your trust-store

> sudo cp ~/quay-install/quay-rootCA/rootCA.pem /etc/pki/ca-trust/source/anchors/quay-rootCA.pem
>
> sudo update-ca-trust

### Authenticate to the redhat.registry.io and your mirror-registry

- Download your pull secret from: https://console.redhat.com/openshift/install/pull-secret
 
> cp ~/Downloads/pull-secret.txt ~/.docker/config.json
>
> or 
>
> cp ~/Downloads/pull-secret.txt $XDG_RUNTIME_DIR/containers/auth.json

- podman login laptop.domain.com with your output from the mirror-registry install

> podman login -u init -p $PASSWORD laptop.kmod.io:8443
>
> -Optional --authfile ~/.docker/config.json

### Launch quay (mirror-registry) in your web browser

> https://laptop.kmod.io:8443

## Populate tue registry for disconnected

In this case we are going to pull the content from the internet and push it directly into the mirror-registry that is running on the laptop. For fully disconnected OpenShift deployment we would modify this step and write the images collected from oc-mirror to local tar files to then move to the disconnected mirror-registry or v2 compatable registry.

### Using oc-mirror

> cat imageset-config.yaml

![imageset-config](images/imageset-config.png)

Populate the laptops registry.

> oc-mirror --config imageset-config.yaml docker://laptop.kmod.io:8443

Save you output, you will need this to build out your install-config as well as to apply to the cluster post deployment. 

![oc-mirror-output](images/oc-mirror-output.png)

You can also find this output in the oc-mirror log file

> cat .oc-mirror

## Time to disconnect from the Internet! 

![network-disconnected](images/network-disconnected-connected.excalidraw.png)

## Configure the openshift install-config and agent-confg

Lets modify the sno-ocp agent-config and install-config to build out our openshift cluster. Two examples are available in the ~/ocp417/ocp/sno-nuc directory for us to work with. 

> cd ~/ocp417/ocp/sno-nuc



### install-config content collection

As we get started in this section I will review the main areas of the install-config that we would like to modify however if you would like to dig a bit deepter in specific sections or taylor the config in other ways you can use the explain function to get additional information. 

> openshift-install explain installconfig
>
> openshift-install explain installconfig.sshKey

To prepare for this process lets get some content outputed to add to the install-config

- If you dont already have an sshkey setup generate one via sshkey-gen or cat out your existing public key. 

> cat ~/.ssh/id_rsa.pub

- Output your mirror-registry ssl cert trust bundle

> cat ~/quay-install/quay-rootCA/rootCA.pem

- Output your oc-mirror imagecontentsourcepolicy (this is how your install points to your mirror-registry for disconnected).

*Note:* your results directory below will be unique to your oc-mirror execution

> cat ~/ocp417/oc-mirror-workspace/results-1732206898/imageContentSourcePolicy.yaml

*Note:* this is the section of the ICSP that we will use for the install-config (release-0)

![imagecontentsourcepolicy](images/imagecontentsourcepolicy.png)

- Output your mirror-registry pull-secret

> cat ~/.docker/config.json

*Note:* we will use the entry with laptop.kmod.io here

![authfile](images/authfile.png)

### install-config modifications

Please use the existing install-config format with your values from the above output. 

![install-config](images/install-config.png)

> vi install-config.yaml

- add your ssh key to the install config in 

> sshKey: ''

- add your sslcert to the install config in

> additionalTrustBundle: |

- add your imagecontentsourcepolicy to the install config in

> imageDigestSources:

- add your mirror-registry pull-secret to the install config in

> pullSecret: FORMAT: '{"auths":{"": {"auth": ""}}}'

### agent-config modifications

The agent config does require some knowledge of the nuc device id's and mount locations. We have multiple ways to caputure this information. The common ones are to boot to a fedora or rhcos instance and inspect them. With the rhcos approach, we could use the default values in this agent config and move to the next step. During the initial boot phases we can inspect the hardware via ssh to the rendezvousIP.

Please use the existing agent-config format with your values

![agent-config](images/agent-config.png)


### Wrapup and backup of the configuration files. 

During the creation of the agent image the openshift-install will consume your agent and install config files. Let make sure that we take a backup copy of them. Add to your favorite source control. 

> mkdir -p bk && cp * bk/

## Create the agent image and write it to the usb-c drive

- create the agent image file

> openshift-install agent create image --log-level debug

*note: * this process will create a auth dir, the agent iso and a rendezvousIP file. We will write the iso to usb and the auth dir contains the info for the kubeadmin user of the cluster. 

- insert your usb key into your laptop

> df -h

It is most comon for the device to show up as /dev/sda 

*Note:* this process will erase everything on the usb key and format it with the agent installer image. 

- write the image to the usb-c drive

> sudo dd if=agent.x86_64.iso of=/dev/sda status=progress

- setup the kubeconfg for authentication on the laptop

> mkdir -p ~/.kube
>
> chmod 600 ~/.kube
>
> cp auth/kubeconfig ~/.kube/config

## Boot the Intel Nuc to the usb drive to install OpenShift

Lets get the usb with the agent install into the nuc and boot it

- hit f12 to boot to the usb-c device

This is the default boot screen for the agent installer on RHCOS
![boot-rhcos](images/boot-rhcos.png)

This is the network, mirror registry, and DNS test screen

*Note:* at this point if you were unsure of the eth device name or the mac address you could enter the network configuration screen to capture it. If this is the case you will need to modify the agent-config.yaml and regenerate the iso, burn the iso, and boot to the new image on the nuc.

![boot-agent-network](images/boot-agent-network.png)

This is the agent waiting for the service
![boot-bootstrap](images/boot-bootstrap.jpg)

Once this step is complete the node will reboot into the new OpenShift image that the agent wrote to the local storage device on the nuc. 

### Interact with the agent and rhcos via cmd

- bootstrap status
> openshift-install agent wait-for bootstrap-complete --log-level debug

- cluster install status
> openshift-install agent wait-for install-complete --log-level debug

- OpenShift Operator status
> watch oc get co

- SSH to rhcos (OpenShift SNO)
> ssh -i ~/.ssh/id_rsa core@192.168.8.199

### OpenShift Cluster Install Complete

- cluster complete from 
> openshift-install agent wait-for install-complete --log-level debug

![cluster-complete](images/cluster-complete.png)

- Copy the url and open the OpenShift console in a web browser. Take note that the kubeadmin password is also in the auth directory as well
> catcat auth/kubeadmin-password

## Post deployment cluster configuration

The last series of steps in this guide we will disable default sources and apply the remaining disconnected artifacts. This will enable the "Operator Hub" with the operators that were mirrored. 

> oc patch OperatorHub cluster --type json -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'
>
> oc apply -f ~/ocp417/oc-mirror-workspace/results-1732206898/imageContentSourcePolicy.yaml
>
> oc apply -f ~/ocp417/oc-mirror-workspace/results-1732206898/catalogSource-cs-redhat-operator-index.yaml
>
>

## to do next
- local storage (LVMs)
>https://docs.openshift.com/container-platform/4.17/storage/persistent_storage/persistent_storage_local/ways-to-provision-local-storage.html#comparison-of-solutions-to-provision-node-local-storage_ways-to-provision-local-storage
> ../../../downloads/butane-amd64 create-a-partition-for-lvmstorage.bu -o 98-create-a-partition-for-lvmstorage.yaml
>https://hackmd.io/@johnsimcall/S1_fuwzyA?utm_source=preview-mode&utm_medium=rec
- ntp
- eval auth file vs podman defaults
- virt operator
- disconnected virt images
- sample operator images
>https://docs.openshift.com/container-platform/4.17/post_installation_configuration/post-install-image-config.html#installation-re[…]all-image-config
- auth
- local registry
>https://docs.openshift.com/container-platform/4.17/registry/configuring_registry_storage/configuring-registry-storage-baremetal.html#configuring-registry-storage-baremetal
