# Loading kernel modules to Openshift nodes using CoreOS layering

### CoreOS layering

Some OpenShift administrators asked for more flexibility for RHEL CoreOS
customization in specific contexts. The RHCOS image layering feature is generally
available in OpenShift 4.13.

CoreOS-Layering is making containers *bootable* in OCP which allows customers to manage
their OS images through containers.
Using containers has become an industry standart and many tools have been developed
arround it including pipelines, security checkers, smoke tests, deliverability and more.
Therefore, allowing *bootable containers* bring a lot of benefits and ease of maintenance
with it.

We can now create a customized base image using a different kernel or any
particular software and consume it in a given cluster.

RHCOS image layering allows you to easily extend the functionality of your base
image by layering additional images onto the base image.
This layering does not modify the base image, and it creates a custom layered
image that includes all base functionality and add-ons to specific nodes in the
cluster.

### Different methods of injecting kernel modules to a cluster

* Day 0: Creating a custom RHCOS image for deployment including the kernel module
* Day 1/2 MCO: The [Machine Config Operator](https://docs.openshift.com/container-platform/4.13/post_installation_configuration/machine-configuration-tasks.html)
    can be used on day2 (as shown in this demo) but can also be used as day1 by adding a
    `MachineConfig` manifests to the cluster installation process (not shown in this demo).
* Day 2 KMM: Using the [Kernel Module Management Operator](https://docs.openshift.com/container-platform/4.13/hardware_enablement/kmm-kernel-module-management.html).

Note: Future recommendations are subject to change as enhancements and features
are added.

### Red Hat support for third party components

![image](https://github.com/ybettan/coreos-layering-driver-toolkit/assets/29724509/b5d494bc-f6c6-41d0-96a7-638fb23e275c)

For more details about the support model, please check the [support-model](https://access.redhat.com/third-party-software-support)
for using third party components such as kernel modules.

### Prerequisites

RHCOS layering is tech preview in OCP 4.12 and fully supported in OCP 4.13+.

We will use the [driver-toolkit](https://github.com/openshift/driver-toolkit)
as a base image in order to build the [simple-kmod](https://github.com/openshift-psap/simple-kmod)
kernel module and then add the `.ko` to a new layer on top of the RHEL-CoreOS
image that runs on the node.

### Get the correct driver-toolkit image

The driver-toolkit contains the kernel package and headers in order to build
kernel modules for a specific kernel, therefore, we need to make sure we use
the correct digest of the driver-toolkit image to fit our specific node.

Let's get the driver-toolkit image for our 4.12.18 cluster
```
$ oc adm release info quay.io/openshift-release-dev/ocp-release:4.12.18-x86_64 --image-for=driver-toolkit
quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:d8f228d47ebf0a5ec0c343f7ed3c5aec1d62d441d671703e5e7f40d78b8d97b9
```

For any other version/arch you can use
```
$ oc adm release info quay.io/openshift-release-dev/ocp-release:<cluste version>-x86_64 --image-for=driver-toolkit
$ oc adm release info quay.io/openshift-release-dev/ocp-release:<cluster version>-aarch64 --image-for=driver-toolkit
```

### Get the correct rhel-coreos-8 image

We want to make sure we are using the same image that runs on the nodes to be
used as the base image of our driver-container image for minimal side effects.
We only want to add our new `.ko` file (and required configuration files if such
exists) to the node. Nothing else should change.

Note:
OCP 4.13 is using `rhel-coreos-9` while OCP 4.12 is using `rhel-coreos-8`.
Be sure to use the correct `--image-for` based on your cluster version.

Let's get the `rhel-coreos` image for our cluster version.
```
$ oc adm release info quay.io/openshift-release-dev/ocp-release:4.12.18-x86_64 --image-for=rhel-coreos-8
quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:127670aaa6073e97bd6d919e1b57bd16b60435410b096aeefce7cd64f6c83d24
```

### Get the correct kernel-version

We also need the correct kernel version in order to build the driver-container.

Since after reboot, the kernel of the host will be the kernel RPM installed on
the new image, aka, the `rhel-coreos-8` image that we are using as the last
layer, then the correct way to get the kernel version is by getting it from the
image itself.
```
$ podman run -it quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:127670aaa6073e97bd6d919e1b57bd16b60435410b096aeefce7cd64f6c83d24 \
    rpm -qa | grep kernel
kernel-modules-4.18.0-372.53.1.el8_6.x86_64
kernel-core-4.18.0-372.53.1.el8_6.x86_64
kernel-4.18.0-372.53.1.el8_6.x86_64
kernel-modules-extra-4.18.0-372.53.1.el8_6.x86_64
```

If you don't have access to the image, make sure you are using your [pull-secret](https://console.redhat.com/openshift/install/pull-secret) with Podman.

We can see the kernel RPM in the image is `4.18.0-372.53.1.el8_6.x86_64`.

### Build the container image

We are going to use the following Dockerfile
```
ARG DTK_IMAGE
ARG RHCOS_IMAGE

FROM ${DTK_IMAGE} as builder
ARG KERNEL_VERSION

WORKDIR /build/

RUN git clone https://github.com/openshift-psap/simple-kmod.git

WORKDIR /build/simple-kmod

RUN make all install KVER=${KERNEL_VERSION}


FROM ${RHCOS_IMAGE}
ARG KERNEL_VERSION

COPY --from=builder /etc/driver-toolkit-release.json /etc/
COPY --from=builder /lib/modules/${KERNEL_VERSION}/*.ko /usr/lib/modules/${KERNEL_VERSION}/

# This is needed in order to autoload the module at boot time.
RUN depmod -a "${KERNEL_VERSION}" && echo simple_kmod > /etc/modules-load.d/simple_kmod.conf
```

For simplicity, let's define some env variables to hold our previous results
```
$ export DTK_IMAGE=quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:d8f228d47ebf0a5ec0c343f7ed3c5aec1d62d441d671703e5e7f40d78b8d97b9
$ export RHCOS_IMAGE=quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:127670aaa6073e97bd6d919e1b57bd16b60435410b096aeefce7cd64f6c83d24
$ export KERNEL_VERSION=4.18.0-372.53.1.el8_6.x86_64
```

Now, we can build the container image.
```
$ podman build \
    --build-arg KERNEL_VERSION=${KERNEL_VERSION} \
    --build-arg DTK_IMAGE=${DTK_IMAGE} \
    --build-arg RHCOS_IMAGE=${RHCOS_IMAGE} \
    -t quay.io/ybettan/coreos-layering:simple-kmod .
[1/2] STEP 1/6: FROM quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:d8f228d47ebf0a5ec0c343f7ed3c5aec1d62d441d671703e5e7f40d78b8d97b9 AS builder
[1/2] STEP 2/6: ARG KERNEL_VERSION
--> Using cache 0a1ec1eb4b3eb2accba3475b2e347ac81c0f01b607d25c5e81c0d01471092d21
--> 0a1ec1eb4b3
[1/2] STEP 3/6: WORKDIR /build/
--> Using cache 02fd2db894dbf7f2c59b7dbabf64c2114eb283ddd230163fb9bd8df892045e02
--> 02fd2db894d
[1/2] STEP 4/6: RUN git clone https://github.com/openshift-psap/simple-kmod.git
--> Using cache 419fda4f52482891200333bfa165ab40d66c40bd4180be1c207dd48ba6fa7d27
--> 419fda4f524
[1/2] STEP 5/6: WORKDIR /build/simple-kmod
--> Using cache 80d50b672deb18ba0f3876e32d487eaaa5832765c87f724c079130576c8cfd6d
--> 80d50b672de
[1/2] STEP 6/6: RUN make all install KVER=4.18.0-372.53.1.el8_6.x86_64
make -C /lib/modules/4.18.0-372.53.1.el8_6.x86_64/build M=/build/simple-kmod EXTRA_CFLAGS=-DKMODVER=\\\"e852852\\\" modules
make[1]: Entering directory '/usr/src/kernels/4.18.0-372.53.1.el8_6.x86_64'
  CC [M]  /build/simple-kmod/simple-kmod.o
  CC [M]  /build/simple-kmod/simple-procfs-kmod.o
  Building modules, stage 2.
  MODPOST 2 modules
  CC      /build/simple-kmod/simple-kmod.mod.o
  LD [M]  /build/simple-kmod/simple-kmod.ko
  CC      /build/simple-kmod/simple-procfs-kmod.mod.o
  LD [M]  /build/simple-kmod/simple-procfs-kmod.ko
make[1]: Leaving directory '/usr/src/kernels/4.18.0-372.53.1.el8_6.x86_64'
gcc -o spkut ./simple-procfs-kmod-userspace-tool.c
install -v -m 755 spkut /bin/
'spkut' -> '/bin/spkut'
install -v -m 755 -d /lib/modules/4.18.0-372.53.1.el8_6.x86_64/
install -v -m 644 simple-kmod.ko        /lib/modules/4.18.0-372.53.1.el8_6.x86_64/simple-kmod.ko
'simple-kmod.ko' -> '/lib/modules/4.18.0-372.53.1.el8_6.x86_64/simple-kmod.ko'
install -v -m 644 simple-procfs-kmod.ko /lib/modules/4.18.0-372.53.1.el8_6.x86_64/simple-procfs-kmod.ko
'simple-procfs-kmod.ko' -> '/lib/modules/4.18.0-372.53.1.el8_6.x86_64/simple-procfs-kmod.ko'
depmod -F /lib/modules/4.18.0-372.53.1.el8_6.x86_64/System.map 4.18.0-372.53.1.el8_6.x86_64
--> 26dfedc51be
[2/2] STEP 1/5: FROM quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:127670aaa6073e97bd6d919e1b57bd16b60435410b096aeefce7cd64f6c83d24
[2/2] STEP 2/5: ARG KERNEL_VERSION
--> 2879805a870
[2/2] STEP 3/5: COPY --from=builder /etc/driver-toolkit-release.json /etc/
--> 8dea5bb2bd8
[2/2] STEP 4/5: COPY --from=builder /lib/modules/4.18.0-372.53.1.el8_6.x86_64/*.ko /usr/lib/modules/4.18.0-372.53.1.el8_6.x86_64/
--> 6249995c281
[2/2] STEP 5/5: RUN depmod -a 4.18.0-372.53.1.el8_6.x86_64 && echo simple_kmod > /etc/modules-load.d/simple_kmod.conf
[2/2] COMMIT quay.io/ybettan/coreos-layering:simple-kmod
--> 612c8c8a54f
Successfully tagged quay.io/ybettan/coreos-layering:simple-kmod
612c8c8a54f0800cf8dcb4c20d5d3d384dc47463711d6d3ad1bd93d52e982b1e
```

Now, let's push the image to some image registry in order for the image to be accesible
by the cluster.
```
$ podman push quay.io/ybettan/coreos-layering:simple-kmod
```

Make sure the image is public or it has the correct pull-secret to pull the image.
The pull secret in the cluster is
```
$ oc get secret/pull-secret -n openshift-config.
```
For more info, check the [how to update the global pull-secret](https://docs.openshift.com/container-platform/4.12/openshift_images/managing_images/using-image-pull-secrets.html#images-update-global-pull-secret_using-image-pull-secrets) doc.

### Loading the kernel module into the nodes

First, we will need to create a `MachineConfig` file.
```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-ybettan-external-image
spec:
  osImageURL: quay.io/ybettan/coreos-layering:simple-kmod
```

Make sure to change the `osImageURL` section in the MachineConfig yaml to the
image you just built, then apply it.
```
$ oc apply -f machineconfig.yaml
```

The Machine Config Operator will start updating workers nodes one by one with
the new image specified in `osImageURL` which contain our kernel module.

Altough there are some optimizations in place to apply live-updates, in some
cases, we will experience a nodes reboot.

This may take some time and you can track the progress using
```
$ watch oc get nodes
```
until all workers nodes are `Ready` again.

Here is a snapshot of the node updating process
```
NAME                           STATUS                     ROLES                  AGE   VERSION
ip-10-0-146-108.ec2.internal   Ready                      control-plane,master   39m   v1.25.8+37a9a08
ip-10-0-147-43.ec2.internal    Ready,SchedulingDisabled   worker                 32m   v1.25.8+37a9a08
ip-10-0-166-128.ec2.internal   Ready                      control-plane,master   39m   v1.25.8+37a9a08
ip-10-0-178-9.ec2.internal     Ready                      worker                 30m   v1.25.8+37a9a08
ip-10-0-226-15.ec2.internal    Ready                      control-plane,master   39m   v1.25.8+37a9a08
ip-10-0-228-190.ec2.internal   Ready                      worker                 29m   v1.25.8+37a9a08
```

For some additional info regarding MCO in case it doesn't update the nodes check
the [machine-config-pull-status](https://docs.openshift.com/container-platform/4.13/post_installation_configuration/machine-configuration-tasks.html#checking-mco-status_post-install-machine-configuration-tasks).

### Validate that the kernel module was loaded

We can start with validating that the node is indeed running a customer image
```
$ oc debug node/ip-10-0-147-43.ec2.internal -- chroot host/ rpm-ostree status
Starting pod/ip-10-0-147-43.ec2.internal-debug ...
To use host binaries, run `chroot /host`
State: idle
Deployments:
* ostree-unverified-registry:quay.io/ybettan/coreos-layering:simple-kmod
                   Digest: sha256:9894d87312b03e04a01371864dd30b56d223f37d7b808ee08f4054eef92f5033
                  Version: 412.86.202305161131-0 (2023-06-04T10:55:24Z)

Removing debug pod ...
```

We can also make sure that the kernel-module was loaded successfuly.
```
$ oc debug node/ip-10-0-147-43.ec2.internal -- chroot host/ lsmod | grep simple_kmod
Starting pod/ip-10-0-147-43ec2internal-debug ...
To use host binaries, run `chroot /host`
simple_kmod            16384  0

Removing debug pod ...
```

# Alternative Solution

For kernel-modules installtion on an OCP cluster, we can also use the dedicated
[kernel-module-management-operator](https://github.com/rh-ecosystem-edge/kernel-module-management)
which is used for day2 kernel-modules installation usecases and some day1 usecases
as well.

# Links

* https://github.com/coreos/coreos-layering-examples
