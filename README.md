# Single Node OpenShift Disconnected on Intel Nuc
![image of networking with nuc](images/network-internet-connected.excalidraw.svg)

In this setup we have the below hardware, take note that the GL wifi router is only connected to the internet during the OCP Bin pull (collect_ocp) and the registry_mirror OCP hydration process

For DNS we will use the GL.iNet wifi router, and for the subnet we will use the default 192.168.8.0/24

* Thinkpad X1 Nano
  * 200gb free space
* Intel Nuc 13 Pro Kit
  * 64 gb DDR4
  * Samsung 2TB nvme
* GL.iNet GL-AXT1800

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

## DNS on GL-AXT1800
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

## Now lets prep the laptop 

This will collect the reqired OpenShift tools to executed a SNO (Single Node OpenShift) install on the Intel Nuc (Disconnected). As well as get the repo and images required for the disconnected install. 

### Clone a base repo to collect the ocp binary, the collect_ocp will download the below files

- oc-mirror
- openshift-install
- openshift-client (OC, kubectl)
- mirror-registry
- butane

> git clone https://github.com/kodonnellredhat/ocp417.git
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

## Now lets populate our registry

In this case we are going to pull the content from the internet and push it directly into the mirror-registry that is running on the laptop. For fully disconnected OpenShift deployment we would modify this step and write the images collected from oc-mirror to local tar files to then move to the disconnected mirror-registry or v2 compatable registry.

### Using oc-mirror

> cat imageset-config.yaml

![imageset-config](images/imageset-config.png)

Populate the laptops registry.

> oc-mirror --config imageset-config.yaml docker://laptop.kmod.io:8443




