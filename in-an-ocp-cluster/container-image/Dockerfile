ARG DTK_IMAGE
ARG RHEL_COREOS_8_IMAGE

FROM ${DTK_IMAGE} as builder
ARG KERNEL_VERSION

WORKDIR /build/

RUN git clone https://github.com/openshift-psap/simple-kmod.git

WORKDIR /build/simple-kmod

RUN make all install KVER=4.18.0-372.26.1.el8_6.x86_64


#FROM quay.io/openshift/origin-rhel-coreos-8:4.12 # mirroring is broken
#FROM registry.ci.openshift.org/ocp/4.12:rhel-coreos-8
FROM ${RHEL_COREOS_8_IMAGE}
ARG KERNEL_VERSION

COPY --from=builder /etc/driver-toolkit-release.json /etc/
COPY --from=builder /lib/modules/${KERNEL_VERSION}/*.ko /usr/lib/modules/${KERNEL_VERSION}/

# This is needed in order to autoload the module at boot time.
RUN depmod -a "${KERNEL_VERSION}" && echo simple_kmod > /etc/modules-load.d/simple_kmod.conf
