# coreos-layering-driver-toolkit

We are going to use the coreos-layering mechanism in order to add additional
layers to the Fedora-CoreOS image.

What it really means, is that the Fedora-CoreOS OS deployed on a machine can
be "layered" the same way as containers are layered.

We are simply going to build a new *container* based on Fedora-CoreOS base
image, add some layers to it and boot a host (not a container) from that newly
created container image.

### Table of content
* [The host is a simple VM running Fedora-CoreOS](./in-a-standalone-fedora-coreos-host)
* [The host is a node in an OCP cluster](./in-an-ocp-cluster)
* [Full blog](./blog)
