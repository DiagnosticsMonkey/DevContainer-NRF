FROM ubuntu:22.04 as base
WORKDIR /workdir
ENV WORKDIR=workdir

ARG sdk_nrf_branch=v2.7-branch
ARG toolchain_version=v2.7.0
ARG sdk_nrf_commit
ARG NORDIC_COMMAND_LINE_TOOLS_VERSION="10-24-0/nrf-command-line-tools-10.24.0"
ARG arch=amd64

# TODO - Drop to user
ARG USERNAME=root

ENV DEBIAN_FRONTEND=noninteractive

SHELL [ "/bin/bash", "-euxo", "pipefail", "-c" ]

RUN <<EOT
    apt-get -y update
    apt-get -y upgrade
    apt-get -y install \
        autoconf \
        automake \
        bison \
        build-essential \
        ca-certificates \
        ccache \
        chrpath \
        cmake \
        cpio \
        device-tree-compiler \
        dfu-util \
        diffstat \
        dos2unix \
        doxygen \
        file \
        flex \
        g++ \
        gawk \
        gcc \
        gcc-multilib \
        gdb \
        git \
        git-core \
        gnupg \
        gperf \
        gtk-sharp2 \
        help2man \
        iproute2 \
        lcov \
        libcairo2-dev \
        libglib2.0-dev \
        libgtk2.0-0 \
        liblocale-gettext-perl \
        libncurses5-dev \
        libpcap-dev \
        libpopt0 \
        libsdl1.2-dev \
        libsdl2-dev \
        libssl-dev \
        libtool \
        libtool-bin \
        locales \
        make \
        net-tools \
        ninja-build \
        openssh-client \
        parallel \
        pkg-config \
        python3-dev \
        python3-pip \
        python3-ply \
        python3-setuptools \
        python3-tk \
        python3-wheel \
        python-is-python3 \
        qemu \
        rsync \
        socat \
        srecord \
        sudo \
        texinfo \
        unzip \
        valgrind \
        wget \
        xz-utils
    
    apt-get -y clean
    rm -rf /var/lib/apt/lists/*
EOT

# Install Python dependencies
RUN pip3 install --no-cache-dir west nrf-regtool>=5.3.0 && \
    pip3 install --no-cache-dir pygobject && \
	pip3 check

# Install toolchain
# Make nrfutil install in a shared location, because when used with GitHub
# Actions, the image will be launched with the home dir mounted from the local
# checkout.
ENV NRFUTIL_HOME=/usr/local/share/nrfutil
RUN <<EOT
    wget -q https://developer.nordicsemi.com/.pc-tools/nrfutil/x64-linux/nrfutil
    mv nrfutil /usr/local/bin
    chmod +x /usr/local/bin/nrfutil
    nrfutil install toolchain-manager
    nrfutil install toolchain-manager search
    nrfutil toolchain-manager install --ncs-version ${toolchain_version}
    nrfutil toolchain-manager list
    rm -f /root/ncs/downloads/*
EOT

# ClangFormat
RUN <<EOT
    wget -qO- https://raw.githubusercontent.com/nrfconnect/sdk-nrf/${sdk_nrf_branch}/.clang-format > /workdir/.clang-format
EOT

# Nordic command line tools
# Releases: https://www.nordicsemi.com/Products/Development-tools/nrf-command-line-tools/download
RUN <<EOT
    NCLT_BASE=https://nsscprodmedia.blob.core.windows.net/prod/software-and-other-downloads/desktop-software/nrf-command-line-tools/sw/versions-10-x-x
    echo "Host architecture: $arch"
    case $arch in
        "amd64")
            NCLT_URL="${NCLT_BASE}/${NORDIC_COMMAND_LINE_TOOLS_VERSION}_linux-amd64.tar.gz"
            ;;
        "arm64")
            NCLT_URL="${NCLT_BASE}/${NORDIC_COMMAND_LINE_TOOLS_VERSION}_linux-arm64.tar.gz"
            ;;
    esac
    echo "NCLT_URL=${NCLT_URL}"
    if [ ! -z "$NCLT_URL" ]; then
        mkdir tmp && cd tmp
        wget -qO - "${NCLT_URL}" | tar --no-same-owner -xz
        # Install included JLink
        mkdir /opt/SEGGER
        tar xzf JLink_*.tgz -C /opt/SEGGER
        mv /opt/SEGGER/JLink* /opt/SEGGER/JLink
        # Install nrf-command-line-tools
        cp -r ./nrf-command-line-tools /opt
        ln -s /opt/nrf-command-line-tools/bin/nrfjprog /usr/local/bin/nrfjprog
        ln -s /opt/nrf-command-line-tools/bin/mergehex /usr/local/bin/mergehex
        cd .. && rm -rf tmp ;
    else
        echo "Skipping nRF Command Line Tools (not available for $arch)" ;
    fi
EOT

# Prepare image with a ready to use build environment
SHELL ["nrfutil","toolchain-manager","launch","/bin/bash","--","-c"]
RUN <<EOT
    west init -m https://github.com/nrfconnect/sdk-nrf --mr ${sdk_nrf_branch} .
    if [[ $sdk_nrf_commit =~ "^[a-fA-F0-9]{32}$" ]]; then
        git checkout ${sdk_nrf_commit};
    fi
    west update --narrow -o=--depth=1
    west zephyr-export
EOT

# Launch into build environment with the passed arguments
# Currently this is not supported in GitHub Actions
# See https://github.com/actions/runner/issues/1964
ENTRYPOINT [ "nrfutil", "toolchain-manager", "launch", "/bin/bash", "--", "/root/entry.sh" ]
COPY ../entry.sh /root/entry.sh

# Clean bash prompt of some hash gore
RUN echo 'PS1="\[\033[01;32m\]\u@docker\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ "' >> ~/.bashrc
# Source on start via bashrc
RUN echo 'source /${WORKDIR}/zephyr/zephyr-env.sh' >> ~/.bashrc

# Add a welcome with info for where Zephyr is...
RUN echo 'echo -e "\033[01;32mWelcome from Diagnostics Monkey\033[00m"' >> ~/.bashrc
RUN echo 'echo -e "\033[01;32mINF: ZEPHYR is located -> ${ZEPHYR_BASE}\033[00m"' >> ~/.bashrc
RUN echo 'echo "     e.g. to build -> \"west build -b nrf5340dk_nrf5340_cpuapp ${ZEPHYR_BASE}/samples/basic/blinky --pristine\""' >> ~/.bashrc

RUN cd /${WORKDIR}/zephyr
RUN west zephyr-export
RUN pip3 install -r /${WORKDIR}/zephyr/scripts/requirements.txt

###

# TODO - Swap for ${sdk_nrf_commit}

# Fix
# CMake Error at /usr/share/cmake-3.22/Modules/FindPackageHandleStandardArgs.cmake:230 (message):
#    Could NOT find Python3 (missing: Interpreter) (Required is at least version
#    "3.8")
#  
#        Reason given by package: 
#            Interpreter: Cannot run the interpreter "/root/ncs/toolchains/e9dba88316/usr/local/bin/python3"
# Debug -> ldd /root/ncs/toolchains/e9dba88316/usr/local/bin/python3, libpython3.9.so.1.0 => not found
ENV LD_LIBRARY_PATH=/root/ncs/toolchains/e9dba88316/usr/local/lib:$LD_LIBRARY_PATH

# Fix 
# CMake Error at /workdir/zephyr/cmake/modules/kernel.cmake:128 (project):
#   Running
#   '/root/ncs/toolchains/e9dba88316/usr/local/bin/ninja' '--version'
#   failed with:
#   /usr/bin/env: 'python3.9': No such file or directory
RUN ln -s /root/ncs/toolchains/e9dba88316/usr/local/bin/python3 /usr/bin/python3.9