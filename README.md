# onload-kmm

This repository describes how to compile and install SolarFlare's OpenOnload kernel modules with Kernel Module Management (KMM) on OpenShift. The integration with the Kernel Module Management (KMM) system is being developed to address the lack of an official AMD solution for building and deploying kernel modules for their Solarflare network cards into OpenShift. This project, which is currently a proof of concept, involves creating a KMM integration that fetches the source RPM from AMD, then compiles, signs, and installs the necessary kernel modules on an OpenShift platform. 

### Platform and Tools  
* OpenShift 4.19.9
* Kernel Module Management 2.4.1
* Node Feature Discovery 4.19.0-*
* OpenOnload SRPM Release Package 9.0.2.140
* Bastion host running RHEL 9.6

## Prerequisites
* OpenShift is installed
* Kernel Module Management operator is installed
* Node Feature Discovery Operator is installed
    * Nodes with Solarflare network cards
    * NFD labels those nodes with `feature.node.kubernetes.io/pci-1924.present=true` as expected
* OpenShift internal registry is [configured](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/registry/setting-up-and-configuring-the-registry) and accessible
* Username/password for user with `cluster-admin` on the OpenShift cluster
* Bastion host running RHEL 9.6
* Red Hat account

## Setup

All instructions should be executed from RHEL bastion host (bastion). 

Variables to be gathered before starting:
* `<cluster_suffix>` - The cluster name and domain. Example: `us-prod-1.ocp.platform.example.com`
* List of: 
    * `<node_name>` and `<node_ip>`
* 

### Prepare local environment

Let's get the local environment ready and check connectivity.  
```shell
git clone https://github.com/openshift-tigerteam/onload-kmm.git
cd onload-kmm  
oc login --server=https://api.<cluster>.<domain>:6443 -u kubeadmin -p <password>
# Login to OpenShift's exposed internal registry (change cluster and domain values)
podman login -u kubeadmin -p $(oc whoami -t) default-route-openshift-image-registry.apps.<cluster_suffix> --tls-verify=false
# Login to registy.redhat.io from local Podman environment
podman login registry.redhat.io -u <user> -p <password>
```
> [!IMPORTANT]
> Disconnect the public remote github and add your own private repo as a remote. 
> There is a .gitignore file to prevent accidental commits of sensitive information. (keys, certs, zip, etc)

## Download the SRPM
* Download [OpenOnload SRPM Release Package](https://www.xilinx.com/support/download/nic-software-and-drivers.html#open) to project root folder.
    * Example: `curl --http2 -O https://www.xilinx.com/content/dam/xilinx/publications/solarflare/onload/openonload/9_0_2_47/sf-122450-ls-17-openonload-srpm-release-package.zip`
* Note the version as `<srpm_version>` for later use. 
    * Example: `9.0.2.140`

## Create OpenShift Project

Create a new project for the onload-kmm work. Create the service account and add the privileged SCC to it. Copy the entitlement certs from `openshift-config-managed` to the new project so that the build process can access the RHEL CRB repository.

```shell
# Create project
oc new-project onload-kmm

# Added privileged RBAC for onload-kmm-sa service account
oc apply -f onload-kmm-sa.yaml -n onload-kmm

# Copy the entitlement certs from openshift-config-managed to onload-kmm project
# This is required to access the RHEL CRB repository during the build process
oc get secret etc-pki-entitlement -n openshift-config-managed -o yaml | \
  sed 's/namespace: openshift-config-managed/namespace: onload-kmm/' | \
  sed '/resourceVersion:/d; /uid:/d; /creationTimestamp:/d' | \
  oc replace -f -
```

## Creating and Importing the Machine Owner Key (MOK) for Onload

Create the keys for the Machine Owner Key (MOK) for Onload and all the formats

```shell
openssl req -new -x509 -newkey rsa:2048 -keyout mok-onload.priv -outform DER -out mok-onload.der -nodes -days 36500 -subj "/CN=OnloadModule/"
openssl x509 -in mok-onload.der -inform DER -out mok-onload.pem -outform PEM
```  

Create secrets using the created keys in onload-kmm project to be used to sign the kernel modules

```shell
oc create secret generic onload-signing-key \
  --from-file=key=mok-onload.priv \
  -n onload-kmm --dry-run=client -o yaml | oc apply -f -

oc create secret generic onload-signing-cert \
  --from-file=cert=mok-onload.pem \
  -n onload-kmm --dry-run=client -o yaml | oc apply -f -
```

> [!IMPORTANT]  
> (START LOOP) Loop over all nodes that will run the onload module and import the MOK key into each node.

Copy the key to the host (DER format) and then import the key into MOK. Do this for every host where the kernel driver needs to be present. 
```shell
scp mok-onload.der core@<node_ip>:/var/home/core/
oc debug node/<node_name>
chroot /host
sudo mokutil --import /var/home/core/mok-onload.der
# Enter password
exit
exit
```

Reboot the host to complete the MOK enrollment. 
```shell
oc adm drain --ignore-daemonsets --delete-emptydir-data <node_name>
oc debug node/<node_name>
chroot /host 
sudo reboot
```

When the host reboots, during the GRUB window, The MOK blue screens will come up. It's on a timer so you have to be quick and hit a key to get into the MOK management screen. Then follow these steps:

* [Shim UEFI Key Management]. Press any key to perform MOK management  
* [Perform MOK Management]. Select `Enroll MOK`  
* [Enroll MOK]. Select `Continue`  
* Enroll the key(s)? Select `Yes`  
* Enroll the key(s)? Enter password  
* [Perform MOK Management]. Select `Reboot`  
* **SYSTEM REBOOTS** 
* Wait for the node to be available. `oc get nodes -w`

After the node reboots and is available, uncordon it. 
```shell
oc debug node/<node_name>
chroot /host 
oc adm uncordon <node_name>
```

> (END LOOP) 

## KMM - Building the SPRM source container image
Build SRPM source container image. The tag should match the version of the onload package downloaded. This will use the `Dockerfile.onload-srpm` dockerfile.
```shell
#Create the build config
oc new-build --name=onload-srpm --strategy=docker --binary -n onload-kmm
# Run the build
tar --transform='s|Dockerfile.onload-srpm|Dockerfile|' \
    -czf - Dockerfile.onload-srpm *.zip \
  | oc start-build onload-srpm --from-archive=- -F -n onload-kmm
# Tag the image to the internal registry with the version of the srpm
oc tag onload-kmm/onload-srpm:latest onload-kmm/onload-srpm:<srpm_version> -n onload-kmm
```

## KMM - The Build

### Create the dockerfile ConfigMap
This ConfigMap contains the dockerfile to be used to compile the module and create the resulting container image containing the `.ko` files. 
```shell
oc apply -f onload-kmm-build-dockerfile.cm.yaml -n onload-kmm
```

## Create the KMM Module CR
This is the KMM module CR which ties everything together to create the build process and the resulting daemonset which signs and installs the `.ko` files onto the identified nodes. 

> [!CAUTION]
> Edit this file to add your version  of the srpm container image created above - `<srpm_version>`. 
> Change the node selector to match your environment. If you are using the `feature.node.kubernetes.io/pci-1924.present=true` label from NFD, make sure that the nodes you want to install the module on have that label.

```shell
oc apply -f onload.module.yaml -n onload-kmm 
```

## Checks

Check that the module is installed and loaded on the nodes.
```shell
oc debug node/<node_name>
chroot /host 
# Show the keys installed
mokutil --list-enrolled
# Is it there?? The big check!
lsmod | egrep 'onload|sfc'
# Example Output
# sh-5.1# lsmod | egrep 'onload|sfc'
# sfc                   688128  0
# mtd                   106496  1 sfc
# onload               1077248  0
# sfc_char              143360  1 onload
# sfc_resource          393216  2 onload,sfc_char
```
The reason you don't see a single sfc module is that the Solarflare network driver has been restructured into multiple, smaller modules for better organization and functionality. The sfc functionality is now distributed across the sfc_char and sfc_resource modules, which you correctly identified as being loaded. This modular design allows OpenOnload to load only the specific driver components it needs, rather than a monolithic sfc module. This is a common practice in modern Linux kernel development to improve maintainability and flexibility. 

### Other Checks
```shell
lspci | grep -i solarflare
lspci -nnk -s <bus:slot.func>
# Example Output
# 03:00.0 Ethernet controller [0200]: Solarflare Communications SFC9220 10G Ethernet [1924:1d22]
#        Subsystem: Solarflare Communications Device [1924:1d22]
#        Kernel driver in use: sfc
#        Kernel modules: sfc, onload
ethtool -i ens1f0
# Example Output
# driver: sfc
# version: 5.14.0-570.35.1.el9_6
# firmware-version: 6.2.3.1000
# bus-info: 0000:03:00.0
# supports-statistics: yes
# supports-test: yes
# supports-eeprom-access: yes
# supports-register-dump: yes
# supports-priv-flags: no
```

List all network interfaces and their drivers
```shell
for dev in $(ls /sys/class/net | grep -v lo); do
  echo "=== $dev ==="
  ethtool -i $dev | grep driver
done
```


### Cleanup Helpers
```shell
oc delete build --all -n onload-kmm
```

## Additional Debugging Notes and Development Items
* The module process to build the specific kmod image is somewhat hard to manage. Specifically, if you *successfully* produce a kmod image with a particular tag but you need to regenerate that image because of a change, the KMM process almost requires you to create a new tag in the `module` - notice the `RE11`. There are some documents on how to force the build to regenerate an image and overwrite the existing tag but haven't had time yet to figure that out. The workaround is to just bump the tag. 
    * The build process also gets completely cleaned up by the operator leaving you with no logs or way to inspect it. That's what that `RUN sleep 10` is for at the end of the build. 
* You can't simply disable the key check using `module.sig_enforce=0`. RHCOS doesn't respect it and forces the practice (which is good).
* This also didn't work. 

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-worker-add-mok-pem
  labels:
    machineconfiguration.openshift.io/role: master
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - path: /etc/pki/ca-trust/source/anchors/MOK.pem
          mode: 0644
          overwrite: true
          contents:
            source: data:text/plain;charset=utf-8;base64,LS0<key_contents>tLS0tCg==
    systemd:
      units:
        - name: update-ca-trust.service
          enabled: true
          contents: |
            [Unit]
            Description=Update system CA trust
            After=network-online.target
            Before=kubelet.service

            [Service]
            Type=oneshot
            ExecStart=/usr/bin/update-ca-trust extract

            [Install]
            WantedBy=multi-user.target
```
* The DTK_AUTO arg in the dockerfile was an attempt to use the DTK container image as the build environment. It didn't work because didn't have the CRB repo enabled and the subscription-manager commands didn't work in that container. You have to use a RHEL or UBI container image and enable the CRB repo.


### Requests for Additions

* How to force rebuild
* require valid tls or push/pull to internal registry, 
  * would require srpc comtainer image pushed to internal tag