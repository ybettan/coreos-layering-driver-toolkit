FROM quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:430be2dbc1b8b547d4639f0ac70817d0f1e7ee20a4e3d70784481a462ba7099a as builder

WORKDIR /build/

RUN git clone https://github.com/openshift-psap/simple-kmod.git

WORKDIR /build/simple-kmod

#FIXME: use a build-arg for the kernel version
RUN make all install KVER=4.18.0-372.19.1.el8_6.x86_64

#FROM quay.io/openshift/origin-rhel-coreos-8:4.12 # mirroring is broken
#FROM registry.ci.openshift.org/ocp/4.12:rhel-coreos-8
FROM quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:edd34e4e74a75099acffe5a15f34b8a7b89fe92810817fdd633b1582f44af4d3

COPY --from=builder /etc/driver-toolkit-release.json /etc/
#FIXME: use a build-arg for the kernel version
COPY --from=builder /lib/modules/4.18.0-372.19.1.el8_6.x86_64/ /usr/lib/modules/4.18.0-372.19.1.el8_6.x86_64/

# This is needed in order to autoload the module at boot time.
RUN echo simple_kmod > /etc/modules-load.d/simple_kmod.conf
