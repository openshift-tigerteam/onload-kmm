# onload-kmm

### Platform and Tools  
* OpenShift 4.19.9
  * Preferred tls certs installed
* Kernel Module Management 2.4.1
* OpenOnload SRPM Release Package 9.0.2.140
* Bastion host running RHEL 9.6

## Prerequisites
* OpenShift is installed
* Kernel Module Management operator is installed
* OpenShift internal registry is [configured](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/registry/setting-up-and-configuring-the-registry) and [exposed](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/registry/securing-exposing-registry) via route
* Username/password for user with `cluster-admin`
* Bastion host running RHEL 9.6
* Red Hat account

## Setup

All instructions should be executed from RHEL bastion host (bastion). 

### Prepare local environment

Let's get the local environment ready and check connectivity.  
```shell
git clone https://github.com/openshift-tigerteam/onload-kmm.git
cd onload-kmm  
oc login --server=https://api.<cluster>.<domain>:6443 -u kubeadmin -p <password>
# Login to OpenShift's exposed internal registry (change cluster and domain values)
podman login -u kubeadmin -p $(oc whoami -t) default-route-openshift-image-registry.apps.<cluster>.<domain> --tls-verify=false
# Login to registy.redhat.io from local Podman environment
podman login registry.redhat.io -u <user> -p <password>
```

## Download the SRPM
* Download [OpenOnload SRPM Release Package](https://www.xilinx.com/support/download/nic-software-and-drivers.html#open) to project root folder. 
`curl --http2 -O https://www.xilinx.com/content/dam/xilinx/publications/solarflare/onload/openonload/9_0_2_47/sf-122450-ls-17-openonload-srpm-release-package.zip`
* Note the version.  

## Copy Certs
If you have any intermediate certs that need to be used for connecting to external repositories, add them to the project root folder as `.pem` files. They will be included in the SRPM source container image in the `/root` folder alongside the SRPM. 

## Create OpenShift Project
```shell
# Create project
oc new-project onload-kmm

# Added privileged RBAC for onload-kmm-sa service account
oc apply -f onload-kmm-sa.yaml -n onload-kmm

oc -n onload-kmm create secret docker-registry kmm-internal-reg \
  --docker-server=image-registry.openshift-image-registry.svc:5000 \
  --docker-username="kubeadmin" \
  --docker-password="$(oc whoami -t)" \
  --docker-email=dummy@example.com

# Copy entitlement to project for dnf access to Red Hat packages
rm -rf /tmp/entitlement && mkdir /tmp/entitlement
## Extract the certs from the source secret
oc extract secret/etc-pki-entitlement -n openshift-config-managed --to=/tmp/entitlement
## Now create a new secret in your target namespace
oc create secret generic etc-pki-entitlement --from-file=/tmp/entitlement -n onload-kmm
```

## Creating and Importing the Machine Owner Key (MOK) for Onload

Create the Machine Owner Key (MOK) for Onload and all the formats
```shell
openssl req -new -x509 -newkey rsa:2048 -keyout mok-onload.priv -outform DER -out mok-onload.der -nodes -days 36500 -subj "/CN=OnloadModule/"
openssl x509 -in mok-onload.der -inform DER -out mok-onload.pem -outform PEM
base64 -w0 mok-onload.pem > mok-onload.pem.b64
```  
Install the MOK into the hosts. Do this for every host where the kernel driver needs to be present. 
```shell
scp mok-onload.der core@<node>:/var/home/core/
oc debug node/<node>
chroot /host
sudo mokutil --import /var/home/core/mok-onload.der
# Enter password
exit
exit
oc adm cordon <node>
oc adm drain --ignore-daemonsets --delete-emptydir-data <node>
oc debug node/<node>
chroot /host 
sudo reboot
```

When the host reboots, during the GRUB window, The MOK blue screens will come up.  
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
oc debug node/<node>
chroot /host 
oc adm uncordon <node>
```

## KMM - Building the SPRM source container image
Build SRPM source container image. The tag should match the version of the onload package downloaded. This will use the  
```shell
podman build -t default-route-openshift-image-registry.apps.<cluster>.<domain>/onload-kmm/onload-srpm:9.0.2.140 -f Containerfile.onload-srpm .
podman push default-route-openshift-image-registry.apps.<cluster>.<domain>/onload-kmm/onload-srpm:9.0.2.140 --tls-verify=false
```

## KMM - The Build
Create secret in onload-kmm project to be used to sign modules
```shell
oc create secret generic onload-signing-keys \
  --from-file=mok-onload.priv \
  --from-file=mok-onload.pem \
  -n onload-kmm --dry-run=client -o yaml | oc apply -f -
```

## Create the dockerfile ConfigMap
This ConfigMap contains the dockerfile to be used to compile the module and create the resulting container image containing the signed `.ko` files. 
```shell
oc apply -f onload-kmm-dockerfile.cm.yaml -n onload-kmm
```

## Create the KMM Module CR
This is the KMM module CR which ties everything together to create the build process and the resulting daemonset which installs the `.ko` files onto the identified nodes. 

> [!CAUTION]
> Edit this file to add your specific registry url

```shell
oc apply -f onload.module.yaml -n onload-kmm 
```
<Add docs about the module, keys, etc>

## Checks
```shell
oc debug node/<node>
chroot /host 
# Show the keys installed
mokutil --list-enrolled
# Is it there?? The big check!
lsmod | egrep 'onload|sfc'
```
#### Check Output
```shell
sh-5.1# lsmod | egrep 'onload|sfc'
onload      Your system is functioning as intended.         1077248  0
sfc_char              143360  1 onload
sfc_resource          393216  2 onload,sfc_char
```
The reason you don't see a single sfc module is that the Solarflare network driver has been restructured into multiple, smaller modules for better organization and functionality. The sfc functionality is now distributed across the sfc_char and sfc_resource modules, which you correctly identified as being loaded. This modular design allows OpenOnload to load only the specific driver components it needs, rather than a monolithic sfc module. This is a common practice in modern Linux kernel development to improve maintainability and flexibility. 

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


### Requests for Additions

* KMM will sign it for you. Can remove from build dockerfile. https://kmm.sigs.k8s.io/documentation/secure_boot/#using-signing-with-kmm
* How to force rebuild
* require valid tls or push/pull to internal registry, 
  * would require srpc comtainer image pushed to internal tag