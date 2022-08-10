# coreos-layering-driver-toolkit

### Build the container image

Let's start with building a simple container image that will contains a basic
golang binary and systemd service in it.

```
cd container-image
podman build -t quay.io/ybettan/coreos-layering:golang-binary .
podman push quay.io/ybettan/coreos-layering:golang-binary
```

### Download the Fedora-CoreOS image

Now, we are going to download the Fedora-CoreOS disk image for Qemu. We
are going to use that disk in order to boot a VM from it later on this
tutorial.

```
curl -L https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/36.20220716.3.1/x86_64/fedora-coreos-36.20220716.3.1-qemu.x86_64.qcow2.xz -o fedora-coreos-36.20220716.3.1-qemu.x86_64.qcow2.xz
```

And extract it.

```
unxz fedora-coreos-36.20220716.3.1-qemu.x86_64.qcow2.xz
```

### Create an ignition file

Ignition files are a way to configure a CoreOS machine at boot time.

```
podman run -i --rm quay.io/coreos/fcct -p -s <fcos-config.fcc > fcos-config.ign
```

# Links
* https://github.com/coreos/coreos-layering-examples
* https://github.com/coreos/coreos-layering-examples
