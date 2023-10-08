# syntax = docker/dockerfile:experimental
FROM ubuntu:22.04 as ovs-builder

ARG ARCH=amd64
ARG NO_AVX512=false
ARG DEBIAN_FRONTEND=noninteractive
ENV SRC_DIR='/usr/src'

RUN apt update && apt install -y git curl build-essential fakeroot \
    autoconf automake bzip2 debhelper-compat dh-exec dh-python dh-sequence-python3 dh-sequence-sphinxdoc \
    graphviz iproute2 libcap-ng-dev libdbus-1-dev libnuma-dev libpcap-dev libssl-dev libtool libunbound-dev \
    openssl pkg-config procps python3-all-dev python3-setuptools python3-sortedcontainers python3-sphinx

RUN cd /usr/src/ && \
    git clone --depth=1 https://github.com/inspurDTest/openvswitch.git && \
    cd openvswitch && \
    chmod a+x boot.sh && \
    chmod a+x debian/rules && \
    chmod a+x debian/openvswitch-switch-dpdk.install && \
    chmod a+x debian/openvswitch-switch.install && \
    chmod a+x build-aux/cksum-schema-check && \
    chmod a+x build-aux/calculate-schema-cksum && \
    ./boot.sh && \
    ./configure && \
    rm -rf .git && \
    CONFIGURE_OPTS='' && \
    if [ "$ARCH" = "amd64" ]; then CONFIGURE_OPTS='CFLAGS="-O2 -g -msse4.2 -mpopcnt -fPIC"'; fi && \
    DATAPATH_CONFIGURE_OPTS='--prefix=/usr' EXTRA_CONFIGURE_OPTS=$CONFIGURE_OPTS DEB_BUILD_OPTIONS='parallel=8 nocheck nodpdk' make debian-deb

RUN dpkg -i /usr/src/python3-openvswitch*.deb

RUN cd /usr/src/ && git clone --depth=1 https://github.com/inspurDTest/ovn-22.09.1.git && \
    cd ovn-22.09.1 && \
    chmod a+x boot.sh && \
    chmod a+x debian/rules && \
    chmod a+x build-aux/cksum-schema-check && \
    chmod a+x build-aux/calculate-schema-cksum && \
    sed -i 's/OVN/ovn/g' debian/changelog && \
    rm -rf .git && \
    ./boot.sh && \
    CONFIGURE_OPTS='--with-ovs-build=/usr/src/openvswitch/_debian' && \
    if [ "$ARCH" = "amd64" ]; then CONFIGURE_OPTS="$CONFIGURE_OPTS CFLAGS='-O2 -g -msse4.2 -mpopcnt -fPIC'"; fi && \
    OVSDIR=/usr/src/openvswitch EXTRA_CONFIGURE_OPTS=$CONFIGURE_OPTS DEB_BUILD_OPTIONS='parallel=8 nocheck' fakeroot debian/rules binary

RUN mkdir /packages/ && \
     # cp /usr/src/libopenvswitch*deb /packages && \
     cp /usr/src/openvswitch-*deb /packages && \
     cp /usr/src/python3-openvswitch*deb /packages && \
     cp /usr/src/ovn-*deb /packages && \
     cd /packages && rm -f *datapath* *docker* *vtep* *test* *dev*

FROM ubuntu:22.04

ARG DEBIAN_FRONTEND=noninteractive
RUN apt update && apt upgrade -y && apt install ca-certificates python3 hostname libunwind8 netbase \
        ethtool iproute2 ncat libunbound-dev procps libatomic1 kmod iptables python3-netifaces python3-sortedcontainers \
        libjs-sphinxdoc tcpdump ipset curl uuid-runtime openssl inetutils-ping arping ndisc6 \
        logrotate dnsutils net-tools strongswan strongswan-pki libcharon-extra-plugins \
        libcharon-extauth-plugins libstrongswan-extra-plugins libstrongswan-standard-plugins -y --no-install-recommends && \
        rm -rf /var/lib/apt/lists/* && \
        rm -rf /etc/localtime

RUN mkdir -p /var/run/openvswitch && \
    mkdir -p /var/run/ovn && \
    mkdir -p /etc/cni/net.d && \
    mkdir -p /opt/cni/bin

ARG ARCH=amd64
ENV CNI_VERSION=v1.2.0
RUN curl -sSf -L --retry 5 https://github.com/containernetworking/plugins/releases/download/${CNI_VERSION}/cni-plugins-linux-amd64-${CNI_VERSION}.tgz | tar -xz -C . ./loopback ./portmap ./macvlan

ENV KUBE_VERSION="v1.28.2"

RUN curl -L https://dl.k8s.io/${KUBE_VERSION}/kubernetes-client-linux-amd64.tar.gz | tar -xz -C . && cp ./kubernetes/client/bin/kubectl /usr/bin/kubectl \
 && chmod +x /usr/bin/kubectl && rm -rf ./kubernetes

RUN --mount=type=bind,target=/packages,from=ovs-builder,source=/packages  \
    # dpkg -i /packages/libopenvswitch*.deb && \
    dpkg -i /packages/python3-openvswitch*.deb && \
    dpkg -i /packages/openvswitch-*.deb && \
    dpkg -i --ignore-depends=openvswitch-switch,openvswitch-common /packages/ovn-*.deb && \
    # mv /usr/lib/python3.7/dist-packages/ovs /usr/lib/python3/dist-packages/ && \
    rm -rf /var/lib/openvswitch/pki/

ENV DUMB_INIT_VERSION="1.2.5"
RUN dump_arch="x86_64"; \
    if [ "$ARCH" = "arm64" ]; then dump_arch="aarch64"; fi; \
    curl -sSf -L --retry 5 -o /usr/bin/dumb-init https://github.com/Yelp/dumb-init/releases/download/v${DUMB_INIT_VERSION}/dumb-init_${DUMB_INIT_VERSION}_${dump_arch} && \
    chmod +x /usr/bin/dumb-init

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

ENTRYPOINT ["/usr/bin/dumb-init", "--"]
