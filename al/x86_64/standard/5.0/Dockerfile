# Copyright 2020-2022 Amazon.com, Inc. or its affiliates. All Rights Reserved.
#
# Licensed under the Amazon Software License (the "License"). You may not use this file except in compliance with the License.
# A copy of the License is located at
#
#    http://aws.amazon.com/asl/
#
# or in the "license" file accompanying this file.
# This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied.
# See the License for the specific language governing permissions and limitations under the License.

FROM public.ecr.aws/amazonlinux/amazonlinux:2023 AS core

# Install SSH, and other utilities
RUN set -ex \
    && yum install -y -q openssh-clients \
    && mkdir ~/.ssh \
    && mkdir -p /opt/tools \
    && mkdir -p /codebuild/image/config \
    && touch ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa,ed25519,ecdsa -H github.com >> ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa,ed25519,ecdsa -H bitbucket.org >> ~/.ssh/known_hosts \
    && chmod 600 ~/.ssh/known_hosts \
    && rpm --import https://download.mono-project.com/repo/xamarin.gpg \
    && curl https://download.mono-project.com/repo/centos7-stable.repo | tee /etc/yum.repos.d/mono-centos7-stable.repo \
    && yum groupinstall -y -q "Development tools" \
    && yum install -y -q --allowerasing \
        ImageMagick acl asciidoc automake binutils bison brotli bzip2-devel \
        coreutils cvs cvsps dnsutils docbook-dtds docbook-style-xsl \
        e2fsprogs expat-devel expect fakeroot file findutils flex \
        g++ gcc git-lfs glib2-devel glibc-all-langpacks glibc-common glibc-langpack-en \
        gnupg2 groff gzip icu iptables jq krb5-server \
        libargon2-devel libcurl-devel libdb-devel libedit-devel libevent-devel libffi-devel \
        libicu-devel libjpeg-devel libpng-devel libsecret-devel libserf libtool libtidy-devel \
        libunwind libwebp-devel libxkbfile-devel libxml2-devel libxslt libxslt-devel \
        libyaml-devel libzip-devel lz4 m4 make mariadb105-devel mercurial mesa-libgbm-devel mlocate \
        net-tools ncurses-devel oniguruma-devel openssl openssl-devel \
        p7zip p7zip-plugins parallel patchelf \
        perl perl-FindBin perl-DBD-SQLite perl-DBI perl-HTTP-Date perl-TimeDate perl-YAML-LibYAML \
        pigz pkg-config postgresql-devel procps-ng python-configobj \
        readline-devel rpm rsync sgml-common sphinx sqlite sqlite-devel sshpass subversion-perl sudo swig \
        tar tcl telnet texinfo time tk tzdata unzip vim wget which  \
        xfsprogs xmlto xorg-x11-server-Xvfb xorriso xz xz-devel zip \
    && rm /etc/yum.repos.d/mono-centos7-stable.repo

# Set locale
ENV LC_ALL="en_US.utf8"
ENV LANG="en_US.utf-8"

RUN useradd codebuild-user

#=======================End of layer: core  =================

FROM core AS tools

# Install AWS SAM CLI
RUN wget -nv https://github.com/aws/aws-sam-cli/releases/latest/download/aws-sam-cli-linux-x86_64.zip -O /tmp/samcli.zip \
    && unzip -q /tmp/samcli.zip -d /opt/samcli \
    && /opt/samcli/install --update -i /usr/local/sam-cli -b /usr/local/bin \
    && rm -rf /opt/samcli /tmp/* \
    && sam --version

# Install AWS CLI v2
# https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html
RUN curl https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o /tmp/awscliv2.zip \
    && unzip -q /tmp/awscliv2.zip -d /opt/awscli \
    && /opt/awscli/aws/install --update -i /usr/local/aws-cli -b /usr/local/bin \
    && rm -rf /opt/awscli /tmp/* \
    && aws --version

# Install Git
RUN set -ex \
    && GIT_VERSION=2.47.1 \
    && GIT_TAR_FILE=git-$GIT_VERSION.tar.gz \
    && GIT_SRC=https://github.com/git/git/archive/v${GIT_VERSION}.tar.gz  \
    && curl -L -o $GIT_TAR_FILE $GIT_SRC \
    && tar zxf $GIT_TAR_FILE \
    && cd git-$GIT_VERSION \
    && make -j4 prefix=/usr \
    && make install prefix=/usr \
    && cd .. && rm -rf git-$GIT_VERSION \
    && rm -rf $GIT_TAR_FILE /tmp/* \
    && git --version

# Install stunnel
RUN set -ex \
    && STUNNEL_VERSION=5.73 \
    && STUNNEL_TAR=stunnel-$STUNNEL_VERSION.tar.gz \
    && STUNNEL_SHA256="bc917c3bcd943a4d632360c067977a31e85e385f5f4845f69749bce88183cb38" \
    && curl -o $STUNNEL_TAR https://www.stunnel.org/archive/5.x/$STUNNEL_TAR && echo "$STUNNEL_SHA256 $STUNNEL_TAR" | sha256sum --check && tar xfz $STUNNEL_TAR \
    && cd stunnel-$STUNNEL_VERSION \
    && ./configure \
    && make -j4 \
    && make install \
    && openssl genrsa -out key.pem 2048 \
    && openssl req -new -x509 -key key.pem -out cert.pem -days 1095 -subj "/C=US/ST=Washington/L=Seattle/O=Amazon/OU=Codebuild/CN=codebuild.amazon.com" \
    && cat key.pem cert.pem >> /usr/local/etc/stunnel/stunnel.pem \
    && cd .. && rm -rf stunnel-${STUNNEL_VERSION}* \
    && stunnel -version

# Install amazon-ecr-credential-helper
# To configure: https://github.com/awslabs/amazon-ecr-credential-helper?tab=readme-ov-file#configuration
# ecr-login is not configured because it conflicts with docker login commands https://github.com/awslabs/amazon-ecr-credential-helper/issues/102
RUN set -ex \
    && yum install -y -q amazon-ecr-credential-helper \
    && docker-credential-ecr-login -v

# Configure SSM
RUN set -ex \
    && yum install -y -q https://s3.amazonaws.com/amazon-ssm-us-east-1/latest/linux_amd64/amazon-ssm-agent.rpm

# Install Pack
RUN set -ex \
    && PACK_VERSION=0.36.0 \
    && (curl -sSL "https://github.com/buildpacks/pack/releases/download/v${PACK_VERSION}/pack-v${PACK_VERSION}-linux.tgz" | tar -C /usr/local/bin/ --no-same-owner -xzv pack) \
    && pack --version

# Railpack
RUN curl -sSL https://railpack.com/install.sh | sh

# Install Docker 27
ENV DOCKER_BUCKET="download.docker.com" \
    DOCKER_CHANNEL="stable" \
    DIND_COMMIT="3b5fac462d21ca164b3778647420016315289034" \
    DOCKER_COMPOSE_VERSION="2.29.7" \
    DOCKER_BUILDX_VERSION="0.17.1"

ENV DOCKER_SHA256="9b4f6fe406e50f9085ee474c451e2bb5adb119a03591f467922d3b4e2ddf31d3" \
    DOCKER_VERSION="27.3.1"

VOLUME /var/lib/docker

RUN set -ex \
    && curl -fSL "https://${DOCKER_BUCKET}/linux/static/${DOCKER_CHANNEL}/x86_64/docker-${DOCKER_VERSION}.tgz" -o docker.tgz \
    && echo "${DOCKER_SHA256} *docker.tgz" | sha256sum -c - \
    && tar --extract --file docker.tgz --strip-components 1  --directory /usr/local/bin/ \
    && rm docker.tgz \
    && docker -v \
    # Check runc version
    && runc -v \
    # Set up subuid/subgid so that "--userns-remap=default" works out-of-the-box
    && groupadd dockremap \
    && useradd -g dockremap dockremap \
    && echo 'dockremap:165536:65536' >> /etc/subuid \
    && echo 'dockremap:165536:65536' >> /etc/subgid \
    && wget -q "https://raw.githubusercontent.com/docker/docker/${DIND_COMMIT}/hack/dind" -O /usr/local/bin/dind \
    # Install docker compose as docker plugin and maintain docker-compose usage
    && mkdir -p /usr/local/lib/docker/cli-plugins \
    && curl -L https://github.com/docker/compose/releases/download/v${DOCKER_COMPOSE_VERSION}/docker-compose-Linux-x86_64 -o /usr/local/lib/docker/cli-plugins/docker-compose \
    && chmod +x /usr/local/lib/docker/cli-plugins/docker-compose \
    && ln -s /usr/local/lib/docker/cli-plugins/docker-compose /usr/local/bin/docker-compose \
    # Ensure docker-compose and docker compose work
    && docker-compose version \
    && docker compose version \
    # Add docker buildx tool
    && curl -L https://github.com/docker/buildx/releases/download/v${DOCKER_BUILDX_VERSION}/buildx-v${DOCKER_BUILDX_VERSION}.linux-amd64 -o /usr/local/lib/docker/cli-plugins/docker-buildx \
    && chmod +x /usr/local/bin/dind /usr/local/lib/docker/cli-plugins/docker-buildx \
    && ln -s /usr/local/lib/docker/cli-plugins/docker-buildx /usr/local/bin/docker-buildx \
    # Ensure docker-buildx works
    && docker-buildx version \
    && docker buildx version \
    # Cleanup
    && rm -rf /tmp/* /var/tmp/*

#=======================End of stage: tools  =================

FROM tools AS runtimes

#****************      NODEJS     *****************************************************

ENV N_SRC_DIR="$SRC_DIR/n"
ENV NODE_18_VERSION="18.20.6" \
    NODE_20_VERSION="20.18.2" \
    NODE_22_VERSION="22.13.1"

RUN git clone https://github.com/tj/n $N_SRC_DIR \
    && cd $N_SRC_DIR && make install

RUN n $NODE_18_VERSION && npm install --save-dev -g -f grunt \
    && npm install --save-dev -g -f grunt-cli \
    && npm install --save-dev -g -f webpack \
    && npm install --save-dev -g -f yarn \
    && n $NODE_20_VERSION && npm install --save-dev -g -f grunt \
    && npm install --save-dev -g -f grunt-cli \
    && npm install --save-dev -g -f webpack \
    && npm install --save-dev -g -f yarn \
    && dnf install -y -v libuv-1.44* \
    && n $NODE_22_VERSION && npm install --save-dev -g -f grunt \
    && npm install --save-dev -g -f grunt-cli \
    && npm install --save-dev -g -f webpack \
    && npm install --save-dev -g -f yarn \
    && cd / && rm -rf $N_SRC_DIR \
    && rm -rf /tmp/* && rm -rf ~/.npm/_logs/

RUN npm install -g npm@10.9.2

# Preserve latest npm version
ENV N_PRESERVE_NPM=1 \
    N_PRESERVE_COREPACK=1
#****************      END NODEJS     **************************************************

#**************** PYTHON ***************************************************************

# Install pyenv
RUN curl -s -S -L https://raw.githubusercontent.com/pyenv/pyenv-installer/master/bin/pyenv-installer | bash
ENV PATH="/root/.pyenv/shims:/root/.pyenv/bin:$PATH"

ENV PYTHON_313_VERSION="3.13.1" \
    PYTHON_311_VERSION="3.11.11" \
    PYTHON_PIP_VERSION="24.3.1" \
    PYYAML_VERSION="6.0.2" \
    PYTHON_SETUPTOOLS_VERSION="75.6.0" \
    PYTHON_CONFIGURE_OPTS="--enable-shared --enable-loadable-sqlite-extensions"

# Python313
RUN set -ex \
    && pyenv install $PYTHON_313_VERSION \
    && pyenv global $PYTHON_313_VERSION \
    && pip3 install --no-cache-dir --upgrade --force-reinstall "pip==$PYTHON_PIP_VERSION" \
    && pip3 install wheel \
    && pip3 install --no-cache-dir --upgrade "setuptools==$PYTHON_SETUPTOOLS_VERSION" boto3 pipenv virtualenv \
    && pip3 install --no-build-isolation "Cython<3" "PyYAML==$PYYAML_VERSION" \
    && pip3 uninstall cython --yes

# Python311
RUN set -ex \
    && pyenv install $PYTHON_311_VERSION \
    && pyenv global $PYTHON_311_VERSION \
    && pip3 install --no-cache-dir --upgrade --force-reinstall "pip==$PYTHON_PIP_VERSION" \
    && pip3 install wheel \
    && pip3 install --no-cache-dir --upgrade "setuptools==$PYTHON_SETUPTOOLS_VERSION" boto3 pipenv virtualenv \
    && pip3 install --no-build-isolation "Cython<3" "PyYAML==$PYYAML_VERSION" \
    && pip3 uninstall cython --yes

#**************** END PYTHON *****************************************************

#****************     GOLANG     ****************************************************

#goenv
RUN git clone https://github.com/syndbg/goenv.git $HOME/.goenv
ENV PATH="/root/.goenv/shims:/root/.goenv/bin:/go/bin:$PATH"

ENV GOLANG_23_VERSION="1.23.4" \
    # GOLANG_22_VERSION="1.22.10" \
    # GOLANG_21_VERSION="1.21.13" 
    GOLANG_20_VERSION="1.20.14"
ENV GOENV_DISABLE_GOPATH=1 \
    GOPATH="/go"

RUN goenv install $GOLANG_23_VERSION \
    # && goenv install $GOLANG_22_VERSION \
    # && goenv install $GOLANG_21_VERSION \
    && goenv install $GOLANG_20_VERSION \
    && goenv global $GOLANG_20_VERSION \
    && go env -w GO111MODULE=auto \
    && go get -u github.com/golang/dep/cmd/dep \
    && rm -rf /tmp/*
#****************      END GOLANG     ****************************************************

FROM runtimes AS al2_v5

# Activate runtime versions specific to image version.
RUN n --preserve $NODE_18_VERSION
RUN pyenv global $PYTHON_311_VERSION
RUN goenv global $GOLANG_20_VERSION

# Configure SSH
COPY ssh_config /root/.ssh/config
COPY runtimes.yml /codebuild/image/config/runtimes.yml
COPY dockerd-entrypoint.sh /usr/local/bin/dockerd-entrypoint.sh
RUN chmod +x /usr/local/bin/dockerd-entrypoint.sh
COPY legal/THIRD_PARTY_LICENSES.txt /usr/share/doc/THIRD_PARTY_LICENSES.txt
COPY legal/bill_of_material.txt /usr/share/doc/bill_of_material.txt
COPY amazon-ssm-agent.json /etc/amazon/ssm/amazon-ssm-agent.json

ENTRYPOINT ["/usr/local/bin/dockerd-entrypoint.sh"]

#=======================End of layer: al2_v5  =================
