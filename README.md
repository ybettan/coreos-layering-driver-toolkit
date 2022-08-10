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

We are going to create a simple ignition file that add your public SSH key to
the machine so you can SSH to it after the installation.

Edit `fcos-config.fcc` and put your public SSH key in it. Then we are going to
generate the ignition file from that yaml.

```
podman run -i --rm quay.io/coreos/fcct -p -s <fcos-config.fcc > fcos-config.ign
```

And make sure it was created correctly by inspecting `fcos-config.ign`.

### Configuring SELinux

We are goign to use `virt` in order to install the VM, therefore, we need to
add a SELinux rule to allow `virt` to read the ignition file.

If you don't want to add any SELInux rule you can temporarly disable it.

* Check SELinux status: `getenforce`
* Disable SELinux: `setenforce 0` (status should become `permissive`)
* Enable SELinux: `setenforce 1` (status should become `enforcing`)

For Adding the relevant rule, we need to

### Install the virtual machine

```
./create-vm
```

### SSH to the machine

Use `virsh net-dhcp-leases default` in order to get the VM IP and then we can SSH to it.

```
ssh core@<ip>
```

### Rebooting from the container

From inside the VM we will use the `rpm-ostree rebase` command in order to
reboot from a spacific container image.

```
rpm-ostree rebase --experimental ostree-unverified-registry:quay.io/ybettan/coreos-layering:golang-binary --bypass-driver
```

### SSH to the new CoreOS image

Once the VM has booted, we can SSH to it again and validate that we indeed have
the "new layer" of the image.

Check the [Dockerfile](container-image/Dockerfile) to understand better what we
are expecting to see in the new VM image.

We can see that the hello-world binary is present.

```
[core@localhost ~]$ hello-world
Hello, world!
```

Also, the systemd service is loaded.

```
[core@localhost ~]$ systemctl status hello-world
○ hello-world.service - A hello world unit!
     Loaded: loaded (/etc/systemd/system/hello-world.service; disabled; vendor preset: disabled)
     Active: inactive (dead)
```


# Links
* https://github.com/coreos/coreos-layering-examples
* https://github.com/coreos/coreos-layering-examples
