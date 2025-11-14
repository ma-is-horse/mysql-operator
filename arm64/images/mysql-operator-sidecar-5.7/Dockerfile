###############################################################################
#  Docker image for sidecar containers
###############################################################################

FROM --platform=$TARGETPLATFORM golang:1.21.13 as builder
USER root
RUN apt update -y
RUN apt install -y wget gcc unzip
ENV GOPATH=/build
WORKDIR /build
#RUN wget -O arm64.zip https://github.com/ksmartdata/mysql-operator/archive/refs/heads/extra_image.zip
RUN wget -O extra_image.zip https://github.com/ma-is-horse/mysql-operator/archive/refs/heads/build-sidecar-5.7.zip
RUN unzip extra_image.zip
WORKDIR /build/mysql-operator-build-sidecar-5.7
RUN CGO_ENABLED=0 go build -o /mysql-operator-sidecar cmd/mysql-operator-sidecar/main.go


FROM --platform=$TARGETPLATFORM centos:8.3.2011 as all
USER root
RUN cd /etc/yum.repos.d/ && sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*  \
    && sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-* \
    && yum makecache
COPY rootfs/ /
RUN yum update -y
RUN yum install -y unzip wget
RUN curl https://rclone.org/install.sh | bash
RUN cp /usr/bin/rclone /usr/local/bin/
RUN chmod +x /usr/local/bin/rclone

# percona-xtrabackup
WORKDIR /build
RUN wget -O percona-xtrabackup-8.0.35-31.zip https://github.com/percona/percona-xtrabackup/archive/refs/tags/percona-xtrabackup-8.0.35-31.zip
RUN unzip percona-xtrabackup-8.0.35-31
WORKDIR /build/percona-xtrabackup-percona-xtrabackup-8.0.35-31
RUN yum install -y \
      cmake \
      openssl-devel \
      libaio libaio-devel \
      automake autoconf \
      bison \
      libtool \
      ncurses-devel \
      libgcrypt-devel \
      libev-devel \
      libcurl-devel \
      zlib-devel \
      vim-common \
      cyrus-sasl-devel \
      libudev-devel \
      gcc-toolset-11-gcc \
      gcc-toolset-11-gcc-c++ \
      gcc-toolset-11-binutils \
    && yum clean all \
    && rm -rf /var/cache/yum

RUN cmake -DWITH_BOOST=PATH-TO-BOOST-LIBRARY -DDOWNLOAD_BOOST=ON \
    -DBUILD_CONFIG=xtrabackup_release -DWITH_MAN_PAGES=OFF -B ..
WORKDIR /build
RUN make -j4
RUN make install

COPY --from=builder /mysql-operator-sidecar /usr/local/bin/
RUN chmod +x /usr/local/bin/mysql-operator-sidecar
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

WORKDIR /build
RUN wget -O percona-toolkit.zip https://github.com/percona/percona-toolkit/archive/refs/heads/3.x.zip
RUN unzip percona-toolkit.zip

WORKDIR /build/percona-toolkit-3.x
RUN yum install perl-ExtUtils-CBuilder perl-ExtUtils-MakeMaker -y
RUN perl Makefile.PL
RUN make
RUN make install


FROM --platform=$TARGETPLATFORM centos:8.3.2011
USER root
RUN cd /etc/yum.repos.d/ \
    && sed -i 's/mirrorlist/#mirrorlist/g' *.repo \
    && sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' *.repo \
    && yum makecache \
    && yum install -y --setopt=tsflags=nodocs \
        perl perl-DBI perl-DBD-MySQL \
    && yum clean all \
    && rm -rf /var/cache/yum
COPY rootfs/ /
COPY --from=all /usr/local/ /usr/local/
COPY --from=all /usr/lib64/ /usr/lib64/

RUN groupmod -g 994 systemd-coredump && \
    groupmod -g 993 input && \
    groupadd -g 999 mysql && \
    useradd -u 999 -r -g 999 -s /sbin/nologin -c "Default Application User" mysql && \
    chown 999:999 \
        /usr/local/bin/docker-entrypoint.sh \
        /usr/local/bin/mysql-operator-sidecar \
        /usr/local/bin/rclone && \
    chmod +x /usr/local/bin/docker-entrypoint.sh

ENV PATH=$PATH:/usr/local/xtrabackup/bin/

ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
