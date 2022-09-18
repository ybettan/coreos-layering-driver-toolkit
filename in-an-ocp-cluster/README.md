# The Fedora-CoreOS host is a node in an OCP cluster

### Prerequisits

This will only work with a later version of RHCOS, currently only working with
OCP 4.12.

Will work for `rpm-ostree` version `2022.10` and above

### Background

This time we are going to build a container image that contains a driver
that we wish to load into the node's kernel.

We will use the [driver-toolkit](https://github.com/openshift/driver-toolkit)
as a base image in order to build the [simple-kmod](https://github.com/openshift-psap/simple-kmod)
driver and then add the driver's binary to a new layer on top of the REHL-CoreOS
image that runs on the node.

### Get the correct driver-toolkit image

The driver-toolkit contains the kernel package in order to building drivers
for a specific kernel, therefore, we need to make sure we use the correct
digest of the driver-toolkit image to fit our specific node.

For a released OCP version:
```
oc adm release info quay.io/openshift-release-dev/ocp-release:<cluste version>-x86_64 --image-for=driver-toolkit
```

For a NON released OCP version:
```
oc adm release info registry.ci.openshift.org/ocp/release:<cluster version> --image-for=driver-toolkit
```

### Get the correct rhel-coreos-8 image

As long as https://issues.redhat.com/browse/OCPBUGS-1035 exists, we need to make
sure to use a base image that won't create any package downgrade so we will look
for the image that fits the most.

For a released OCP version:
```
oc adm release info quay.io/openshift-release-dev/ocp-release:<cluster version> --image-for=rhel-coreos-8
```

For a NON released OCP version:
```
oc adm release info registry.ci.openshift.org/ocp/release:<cluster version> --image-for=rhel-coreos-8
```

Once the issue is resolved, in theory, even images versions that creates packages
downgrade are supposed to work.

### Build the container image

```
cd in-an-ocp-cluster/container-image/
#FIXME: add the kernel version as a build-arg
podman build -t quay.io/ybettan/coreos-layering:simple-kmod .
podman push quay.io/ybettan/coreos-layering:simple-kmod
```

### Rebooting from the container

Make sure to change the `osImageURL` section in the MachineConfig yaml, then
apply it.
```
oc apply -f in-an-ocp-cluster/machineconfig.yaml
```

### Validate that the kernel module was loaded

Get into the node using `oc debug node/<node name>` followed by `chroot /host`
to get the host binaries.

Then check if the `simple_kmod` kernel module was loaded.
```
lsmod | grep simple_kmod
```
