# Docker file for an EverGreenCoin node
# This is based on Debian 11 and will compile against Berkley DB 4.8

# Common base with things we always need
ARG DIST=debian
ARG DIST_VER=11
ARG BOOST_VER=1.74

FROM ${DIST}:${DIST_VER} AS base
ARG TERM=xterm
ENV TZ=Etc/UTC

RUN apt-get update && apt-get install -y --no-install-recommends \
	apt-utils \
	dialog \
	perl-modules \
	&& rm -rf /var/lib/apt/lists/*

# Builder image to build the bits we need - This can be removed later
FROM base AS builder
ARG TERM=xterm
ARG BUILD_BASE=/home/evergreencoin
ARG DEBIAN_FRONTEND=noninteractive
# TODO we might only need some of the libboost libs but this is easier and temporary
RUN apt-get update && apt-get install -y --no-install-recommends \
	build-essential \
	ca-certificates \
	git \
	libboost-all-dev \
	libssl-dev \
	libminiupnpc-dev \
	libcurl4-openssl-dev \
	zlib1g-dev \
	&& rm -rf /var/lib/apt/lists/*

# Add the Berkly DB source
ARG BDB_VERSION=db-4.8.30.NC
WORKDIR ${BUILD_BASE}
ADD ${BDB_VERSION}.tar.gz ${BUILD_BASE}/

# Apply a patch necessary when building with clang and c++11 (see https://community.oracle.com/thread/3952592)
WORKDIR ${BUILD_BASE}/${BDB_VERSION}
ADD db-4.8.30-atomic.patch ../${BDB_VERSION}-atomic.patch
RUN patch -p1 < ../${BDB_VERSION}-atomic.patch

# Build and Install
WORKDIR ${BUILD_BASE}/${BDB_VERSION}/build_unix
RUN ../dist/configure --enable-cxx --disable-shared --with-pic --prefix=/usr/local
RUN make
# If we don't want the DB tools we could skip the `make install` and tell
# the EGC makefile where to find the includes/libraries
RUN make install

# Get the EverGreenCoin Sources
ARG EGC_VERSION=v1.9.5.2
WORKDIR ${BUILD_BASE}
RUN git clone https://github.com/EverGreenCoinDev/EverGreenCoin.git --branch ${EGC_VERSION} --single-branch

# Build and Install
WORKDIR ${BUILD_BASE}/EverGreenCoin/src
RUN make -f makefile.unix -e PIE=1
# These Env vars are only needed if we didn't install Berkely DB
# -e BDB_INCLUDE_PATH=${BUILD_BASE}/${BDB_VERSION}/build_unix -e BDB_LIB_PATH=${BUILD_BASE}/${BDB_VERSION}/build_unix
RUN cp evergreencoind /usr/local/bin/

# Strip
RUN strip /usr/local/bin/evergreencoind

FROM base AS evergreencoin
ARG TERM=xterm
ARG BOOST_VER
ARG DATADIR=/var/lib/evergreencoin

LABEL maintainer="Mike Axford <mfaxford@gmail.com>"
ENV LANG=C.UTF-8

# Create User
RUN useradd -r -c "EverGreenCoin Wallet Daemon" -d ${DATADIR} -s /usr/sbin/nologin evergreencoin \
	&& mkdir ${DATADIR} \
	&& chown evergreencoin: ${DATADIR} \
	&& chmod 750 ${DATADIR}

# Distribution sepecific packages
RUN apt-get update && apt-get install -y --no-install-recommends \
	libboost-system${BOOST_VER} \
	libboost-filesystem${BOOST_VER} \
	libboost-program-options${BOOST_VER} \
	libboost-thread${BOOST_VER} \
	libboost-chrono${BOOST_VER} \
	libboost-date-time${BOOST_VER} \
	libminiupnpc17 \
	libssl1.1 \
	logrotate \
	nano-tiny \
	psmisc \
	vim-tiny \
	xz-utils \
	&& rm -rf /var/lib/apt/lists/*

COPY --from=builder /usr/local/bin /usr/local/bin
COPY --from=builder /usr/local/include /usr/local/include
COPY --from=builder /usr/local/lib /usr/local/lib
COPY --chown=root:root evergreencoin.logrotate /etc/logrotate.d/evergreencoin
COPY --chown=evergreencoin:evergreencoin evergreencoin.conf ${DATADIR}
RUN mkdir ${DATADIR}/.evergreencoin \
	&& cd ${DATADIR}/.evergreencoin \
	&& ln -s ../evergreencoin.conf

# Have a folder in which to store our data
VOLUME ${DATADIR}

# 5757 is the network port - Ideally this should be open for external connections
# 5758 is the RPC port so should only be talked to locally
EXPOSE 5757/tcp 5758/tcp

USER evergreencoin
WORKDIR ${DATADIR}

# HEALTHCHECK [options] CMD command
# --interval=30s
#--timeout=30s
#--start-period=0s
#--retries=3

#HEALTHCHECK --interval=5m --timeout=10s CMD ["evergreencoind", "-datadir=${DATADIR}", "getinfo", "|| exit 1"]

# TODO Have ENTRYPOINT use a script that can do some initial setup (set RPC User/Password etc)
ENTRYPOINT ["evergreencoind", "-datadir=/var/lib/evergreencoin"]
#CMD
# Set an RPC Password if not specified
# openssl rand -base64 12


# Add to *base* if we want a different language for the environment
#RUN apt-get update && apt-get install -y apt-utils locales && rm -rf /var/lib/apt/lists/* \
    #&& localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8
#ENV LANG en_US.utf8

# If we want to download the db source code rather than use a local copy in *builder*
#ADD http://download.oracle.com/berkeley-db/${BDB_VERSION}.tar.gz /home/evergreencoin
#RUN tar -xzf ${BDB_VERSION}.tar.gz

# Extra options for building Berkley DB - any use to add ?
# --disable-replication --enable-option-checking
