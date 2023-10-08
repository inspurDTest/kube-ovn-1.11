# syntax = docker/dockerfile:experimental
FROM ubuntu:22.04 as ovs-builder

ARG ARCH
ARG DEBIAN_FRONTEND=noninteractive
ENV SRC_DIR='/usr/src'

RUN apt update && apt install -y git curl

RUN cd /usr/src/ && git clone https://github.com/inspurDTest/openvswitch.git && mv openvswitch ovs

RUN cd /usr/src/ && git clone https://github.com/inspurDTest/ovn-22.09.1.git && mv ovn-22.09.1 ovn

RUN apt install -y build-essential fakeroot \
    autoconf automake bzip2 debhelper-compat dh-exec dh-python dh-sequence-python3 dh-sequence-sphinxdoc \
    graphviz iproute2 libcap-ng-dev libdbus-1-dev libnuma-dev libpcap-dev libssl-dev libtool libunbound-dev \
    openssl pkg-config procps python3-all-dev python3-setuptools python3-sortedcontainers python3-sphinx

RUN cd /usr/src/ovs && \
    chmod a+x boot.sh && \
    chmod a+x debian/rules && \
    chmod a+x debian/openvswitch-switch.install && \
    chmod a+x debian/openvswitch-switch-dpdk.install && \
    ./boot.sh && \
    ./configure && \
    rm -rf .git && \
    CONFIGURE_OPTS='' && \
    if [ "$ARCH" = "amd64" ]; then CONFIGURE_OPTS='CFLAGS="-O2 -g -msse4.2 -mpopcnt -fPIC"'; fi && \
    DATAPATH_CONFIGURE_OPTS='--prefix=/usr' EXTRA_CONFIGURE_OPTS=$CONFIGURE_OPTS DEB_BUILD_OPTIONS='parallel=8 nocheck nodpdk' make debian-deb

RUN dpkg -i /usr/src/python3-openvswitch*.deb

RUN cd /usr/src/ovn && \
    chmod a+x boot.sh && \
    sed -i 's/OVN/ovn/g' debian/changelog && \
    rm -rf .git && \
    ./boot.sh && \
    CONFIGURE_OPTS='--with-ovs-build=/usr/src/ovs/_debian' && \
    if [ "$ARCH" = "amd64" ]; then CONFIGURE_OPTS="$CONFIGURE_OPTS CFLAGS='-O2 -g -msse4.2 -mpopcnt -fPIC'"; fi && \
    OVSDIR=/usr/src/ovs EXTRA_CONFIGURE_OPTS=$CONFIGURE_OPTS DEB_BUILD_OPTIONS='parallel=8 nocheck' fakeroot debian/rules binary

RUN mkdir /packages/ && \
    cp /usr/src/openvswitch-*deb /packages && \
    cp /usr/src/python3-openvswitch*deb /packages && \
    cp /usr/src/ovn-*deb /packages && \
    cd /packages && rm -f *source* *doc* *datapath* *docker* *vtep* *test* *dev*

FROM ubuntu:22.04

ARG DEBIAN_FRONTEND=noninteractive
RUN apt update && apt upgrade -y && apt install ca-certificates python3 hostname libunwind8 netbase \
        ethtool iproute2 ncat libunbound-dev procps libatomic1 kmod iptables python3-netifaces python3-sortedcontainers \
        tcpdump ipset curl uuid-runtime openssl inetutils-ping arping ndisc6 \
        logrotate dnsutils net-tools strongswan strongswan-pki libcharon-extra-plugins \
        libcharon-extauth-plugins libstrongswan-extra-plugins libstrongswan-standard-plugins -y --no-install-recommends && \
        rm -rf /var/lib/apt/lists/* && \
        rm -rf /etc/localtime

RUN mkdir -p /var/run/openvswitch && \
    mkdir -p /var/run/ovn && \
    mkdir -p /etc/cni/net.d && \
    mkdir -p /opt/cni/bin

ARG ARCH
ENV CNI_VERSION=v1.3.0
RUN curl -sSf -L --retry 5 https://github.com/containernetworking/plugins/releases/download/${CNI_VERSION}/cni-plugins-linux-${ARCH}-${CNI_VERSION}.tgz | tar -xz -C . ./loopback ./portmap ./macvlan

ENV KUBE_VERSION="v1.28.2"

RUN curl -L https://dl.k8s.io/${KUBE_VERSION}/kubernetes-client-linux-${ARCH}.tar.gz | tar -xz -C . && cp ./kubernetes/client/bin/kubectl /usr/bin/kubectl \
 && chmod +x /usr/bin/kubectl && rm -rf ./kubernetes

ENV BFDD_VERSION="v0.5.4"
RUN curl -sSf -L --retry 3 -o /usr/local/bin/bfdd-control https://github.com/bobz965/bfd-binary-for-kube-ovn-cni/releases/download/${BFDD_VERSION}/bfdd-control && \
    curl -sSf -L --retry 3 -o /usr/local/bin/bfdd-beacon https://github.com/bobz965/bfd-binary-for-kube-ovn-cni/releases/download/${BFDD_VERSION}/bfdd-beacon && \
    chmod +x /usr/local/bin/bfdd-control /usr/local/bin/bfdd-beacon

RUN --mount=type=bind,target=/packages,from=ovs-builder,source=/packages  \
    dpkg -i /packages/openvswitch-*.deb /packages/python3-openvswitch*.deb && \
    dpkg -i --ignore-depends=openvswitch-switch,openvswitch-common /packages/ovn-*.deb && \
    rm -rf /var/lib/openvswitch/pki/

ARG DEBUG=false
RUN --mount=type=bind,target=/packages,from=ovs-builder,source=/packages \
    if [ "${DEBUG}" = "true" ]; then \
        apt update && apt install -y --no-install-recommends gdb valgrind && \
        rm -rf /var/lib/apt/lists/* && \
        dpkg -i --ignore-depends=openvswitch-switch,openvswitch-common /packages/*.ddeb; \
    fi
