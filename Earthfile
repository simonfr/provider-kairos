VERSION 0.6

IMPORT github.com/kairos-io/kairos

FROM alpine
ARG VARIANT=kairos # core, lite, framework
ARG FLAVOR=opensuse-leap

## Versioning
ARG K3S_VERSION
RUN apk add git
COPY . ./
RUN echo $(git describe --always --tags --dirty) > VERSION
ARG CORE_VERSION=$(cat CORE_VERSION || echo "latest")
ARG VERSION=$(cat VERSION)
RUN echo "version ${VERSION}"
ARG K3S_VERSION_TAG=$(echo $K3S_VERSION | sed s/+/-/)
ARG TAG=${VERSION}-k3s${K3S_VERSION_TAG}
ARG BASE_REPO=quay.io/kairos
ARG IMAGE=${BASE_REPO}/${VARIANT}-${FLAVOR}:$TAG
ARG BASE_IMAGE=quay.io/kairos/core-${FLAVOR}:${CORE_VERSION}
ARG ISO_NAME=${VARIANT}-${FLAVOR}-${VERSION}-k3s${K3S_VERSION}
# renovate: datasource=docker depName=quay.io/kairos/osbuilder-tools versioning=semver-coerced
ARG OSBUILDER_VERSION=v0.7.10
ARG OSBUILDER_IMAGE=quay.io/kairos/osbuilder-tools:$OSBUILDER_VERSION

## External deps pinned versions
ARG LUET_VERSION=0.33.0
ARG GOLANGCILINT_VERSION=v1.52-alpine
ARG HADOLINT_VERSION=2.12.0-alpine
ARG SHELLCHECK_VERSION=v0.9.0
# renovate: datasource=docker depName=golang
ARG GO_VERSION=1.20

ARG OS_ID=kairos
ARG CGO_ENABLED=0

RELEASEVERSION:
    COMMAND
    RUN echo "$IMAGE" > IMAGE
    RUN echo "$VERSION" > VERSION
    SAVE ARTIFACT VERSION AS LOCAL build/VERSION
    SAVE ARTIFACT IMAGE AS LOCAL build/IMAGE

all-arm-generic:
  BUILD --platform=linux/arm64 +docker
  BUILD --platform=linux/arm64 +image-sbom
  BUILD --platform=linux/arm64 +iso
  DO +RELEASEVERSION

all:
  ARG SECURITY_SCANS=true
  BUILD +docker
  IF [ "$SECURITY_SCANS" = "true" ]
      BUILD +image-sbom
  END
  BUILD +iso
  BUILD +netboot
  BUILD +ipxe-iso
  DO +RELEASEVERSION

ci:
  BUILD +docker
  BUILD +iso

all-arm:
  ARG SECURITY_SCANS=true
  BUILD --platform=linux/arm64 +docker
  IF [ "$SECURITY_SCANS" = "true" ]
      BUILD --platform=linux/arm64  +image-sbom
  END
  BUILD +arm-image --MODEL=rpi64
  DO +RELEASEVERSION

go-deps:
    ARG GO_VERSION
    FROM golang:$GO_VERSION
    WORKDIR /build
    COPY go.mod go.sum ./
    RUN go mod download
    SAVE ARTIFACT go.mod AS LOCAL go.mod
    SAVE ARTIFACT go.sum AS LOCAL go.sum

test:
    FROM +go-deps
    WORKDIR /build
    RUN go install -mod=mod github.com/onsi/ginkgo/v2/ginkgo
    COPY (kairos+luet/luet) /usr/bin/luet
    COPY . .
    RUN ginkgo run --fail-fast --slow-spec-threshold 30s --covermode=atomic --coverprofile=coverage.out -p -r ./internal
    SAVE ARTIFACT coverage.out AS LOCAL coverage.out

BUILD_GOLANG:
    COMMAND
    WORKDIR /build
    COPY . ./
    ARG CGO_ENABLED
    ARG VERSION
    ARG LDFLAGS="-s -w -X 'github.com/kairos-io/provider-kairos/v2/internal/cli.VERSION=$VERSION'"
    ARG BIN
    ARG SRC
    ENV CGO_ENABLED=${CGO_ENABLED}
    RUN echo $LDFLAGS
    RUN go build -ldflags "${LDFLAGS}" -o ${BIN} ${SRC}
    SAVE ARTIFACT ${BIN} ${BIN} AS LOCAL build/${BIN}

build-kairos-agent-provider:
    FROM +go-deps
    DO +BUILD_GOLANG --BIN=agent-provider-kairos --SRC=./ --CGO_ENABLED=$CGO_ENABLED

build-kairosctl:
    FROM +go-deps
    DO +BUILD_GOLANG --BIN=kairosctl --SRC=./cli/kairosctl --CGO_ENABLED=$CGO_ENABLED

build:
    BUILD +build-kairos-agent-provider
    BUILD +build-kairosctl

version:
    FROM alpine
    RUN apk add git

    COPY . ./

    RUN --no-cache echo $(git describe --always --tags --dirty) > VERSION

    ARG VERSION=$(cat VERSION)
    SAVE ARTIFACT VERSION VERSION

dist:
    ARG GO_VERSION
    FROM golang:$GO_VERSION
    RUN echo 'deb [trusted=yes] https://repo.goreleaser.com/apt/ /' | tee /etc/apt/sources.list.d/goreleaser.list
    RUN apt update
    RUN apt install -y goreleaser
    WORKDIR /build
    COPY . .
    COPY +version/VERSION ./
    RUN echo $(cat VERSION)
    RUN VERSION=$(cat VERSION) goreleaser build --rm-dist --skip-validate --snapshot
    SAVE ARTIFACT /build/dist/* AS LOCAL dist/

docker:
    ARG FLAVOR
    ARG VARIANT
    FROM $BASE_IMAGE

    DO +PROVIDER_INSTALL

    ARG KAIROS_VERSION
    IF [ "$KAIROS_VERSION" = "" ]
        ARG OS_VERSION=${VERSION}
    ELSE
        ARG OS_VERSION=${KAIROS_VERSION}
    END

    ARG OS_ID
    ARG OS_NAME=${OS_ID}-${FLAVOR}
    ARG OS_REPO=quay.io/kairos/${VARIANT}-${FLAVOR}
    ARG OS_LABEL=latest

    DO kairos+OSRELEASE --BUG_REPORT_URL="https://github.com/kairos-io/kairos/issues/new/choose" --HOME_URL="https://github.com/kairos-io/provider-kairos" --OS_ID=${OS_ID} --OS_LABEL=${OS_LABEL} --OS_NAME=${OS_NAME} --OS_REPO=${OS_REPO} --OS_VERSION=${OS_VERSION}-k3s${K3S_VERSION} --GITHUB_REPO="kairos-io/provider-kairos" --VARIANT=${VARIANT}

    SAVE IMAGE $IMAGE

# This install the requirements for the provider to be included.
# Made as a command so it can be reused from other targets without depending on this repo BASE_IMAGE
PROVIDER_INSTALL:
    COMMAND
    IF [ "$K3S_VERSION" = "latest" ]
    ELSE
        ENV INSTALL_K3S_VERSION=${K3S_VERSION}
    END

    IF [ "$FLAVOR" = "opensuse-leap" ] || [ "$FLAVOR" = "opensuse-leap-arm-rpi" ]
      RUN zypper ref && zypper in -y nohang
    ELSE IF [ "$FLAVOR" = "alpine-ubuntu" ] || [ "$FLAVOR" = "alpine-opensuse-leap" ] || [ "$FLAVOR" = "alpine-arm-rpi" ]
      RUN apk add grep
    ELSE IF [ "$FLAVOR" = "opensuse-tumbleweed" ] || [ "$FLAVOR" = "opensuse-tumbleweed-arm-rpi" ]
      RUN zypper ref && zypper in -y nohang
    ELSE IF [ "$FLAVOR" = "ubuntu" ] || [ "$FLAVOR" = "ubuntu-20-lts" ] || [ "$FLAVOR" = "ubuntu-22-lts" ] || [ "$FLAVOR" = "debian" ]
      RUN apt-get update && apt-get install -y nohang
    END

    ENV INSTALL_K3S_BIN_DIR="/usr/bin"
    RUN curl -sfL https://get.k3s.io > installer.sh \
        && INSTALL_K3S_SELINUX_WARN=true INSTALL_K3S_SKIP_START="true" INSTALL_K3S_SKIP_ENABLE="true" INSTALL_K3S_SKIP_SELINUX_RPM="true" bash installer.sh \
        && INSTALL_K3S_SELINUX_WARN=true INSTALL_K3S_SKIP_START="true" INSTALL_K3S_SKIP_ENABLE="true" INSTALL_K3S_SKIP_SELINUX_RPM="true" bash installer.sh agent \
        && rm -rf installer.sh

    # If base image does not bundle a luet config use one
    # TODO: Remove this, use luet config from base images so they are in sync
    IF [ ! -e "/etc/luet/luet.yaml" ]
        COPY repository.yaml /etc/luet/luet.yaml
    END

    RUN luet install -y utils/edgevpn utils/k9s utils/nerdctl container/kubectl utils/kube-vip && luet cleanup
    # Drop env files from k3s as we will generate them
    IF [ -e "/etc/rancher/k3s/k3s.env" ]
        RUN rm -rf /etc/rancher/k3s/k3s.env /etc/rancher/k3s/k3s-agent.env && touch /etc/rancher/k3s/.keep
    END

    COPY +build-kairos-agent-provider/agent-provider-kairos /system/providers/agent-provider-kairos
    RUN ln -s /system/providers/agent-provider-kairos /usr/bin/kairos


docker-rootfs:
    FROM +docker
    SAVE ARTIFACT /. rootfs

kairos:
   ARG KAIROS_VERSION=master
   FROM alpine
   RUN apk add git
   WORKDIR /kairos
   RUN git clone https://github.com/kairos-io/kairos /kairos && cd /kairos && git checkout "$KAIROS_VERSION"
   SAVE ARTIFACT /kairos/

get-kairos-scripts:
    FROM alpine
    WORKDIR /build
    COPY +kairos/kairos/ ./
    SAVE ARTIFACT /build/scripts AS LOCAL scripts

iso:
    ARG OSBUILDER_IMAGE
    ARG ISO_NAME=${OS_ID}
    ARG IMG=docker:$IMAGE
    ARG overlay=overlay/files-iso
    FROM $OSBUILDER_IMAGE
    RUN zypper in -y jq docker
    WORKDIR /build
    COPY . ./
    RUN mkdir -p overlay/files-iso
    COPY +kairos/kairos/overlay/files-iso/ ./$overlay/
    COPY +docker-rootfs/rootfs /build/image
    RUN /entrypoint.sh --name $ISO_NAME --debug build-iso --date=false dir:/build/image --overlay-iso /build/${overlay} --output /build
    # See: https://github.com/rancher/elemental-cli/issues/228
    RUN sha256sum $ISO_NAME.iso > $ISO_NAME.iso.sha256
    SAVE ARTIFACT /build/$ISO_NAME.iso kairos.iso AS LOCAL build/$ISO_NAME.iso
    SAVE ARTIFACT /build/$ISO_NAME.iso.sha256 kairos.iso.sha256 AS LOCAL build/$ISO_NAME.iso.sha256

netboot:
   FROM opensuse/leap
   ARG VERSION
   ARG ISO_NAME
   WORKDIR /build
   COPY +iso/kairos.iso kairos.iso
   COPY . .
   RUN zypper in -y cdrtools

   COPY +kairos/kairos/scripts/netboot.sh ./
   RUN sh netboot.sh kairos.iso $ISO_NAME $VERSION

   SAVE ARTIFACT /build/$ISO_NAME.squashfs squashfs AS LOCAL build/$ISO_NAME.squashfs
   SAVE ARTIFACT /build/$ISO_NAME-kernel kernel AS LOCAL build/$ISO_NAME-kernel
   SAVE ARTIFACT /build/$ISO_NAME-initrd initrd AS LOCAL build/$ISO_NAME-initrd
   SAVE ARTIFACT /build/$ISO_NAME.ipxe ipxe AS LOCAL build/$ISO_NAME.ipxe

arm-image:
  ARG OSBUILDER_IMAGE
  ARG COMPRESS_IMG=true
  FROM $OSBUILDER_IMAGE
  ARG MODEL=rpi64
  ARG IMAGE_NAME=${VARIANT}-${FLAVOR}-${VERSION}-k3s${K3S_VERSION}.img
  WORKDIR /build

  ENV SIZE="15200"

  IF [[ "$FLAVOR" = "ubuntu-20-lts-arm-nvidia-jetson-agx-orin" ]]
    ENV STATE_SIZE="14000"
    ENV RECOVERY_SIZE="10000"
    ENV DEFAULT_ACTIVE_SIZE="4500"
  ELSE IF [[ "$FLAVOR" =~ ^ubuntu* ]]
    ENV STATE_SIZE="6900"
    ENV RECOVERY_SIZE="4600"
    ENV DEFAULT_ACTIVE_SIZE="2700"
  ELSE
    ENV STATE_SIZE="6200"
    ENV RECOVERY_SIZE="4200"
    ENV DEFAULT_ACTIVE_SIZE="2000"
  END


  COPY --platform=linux/arm64 +docker-rootfs/rootfs /build/image
  # With docker is required for loop devices
  WITH DOCKER --allow-privileged
    RUN /build-arm-image.sh --use-lvm --model $MODEL --directory "/build/image" /build/$IMAGE_NAME
  END
  IF [ "$COMPRESS_IMG" = "true" ]
    RUN xz -v /build/$IMAGE_NAME
    SAVE ARTIFACT /build/$IMAGE_NAME.xz img AS LOCAL build/$IMAGE_NAME.xz
  ELSE
    SAVE ARTIFACT /build/$IMAGE_NAME img AS LOCAL build/$IMAGE_NAME
  END
  SAVE ARTIFACT /build/$IMAGE_NAME.sha256 img-sha256 AS LOCAL build/$IMAGE_NAME.sha256

syft:
    FROM anchore/syft:latest
    SAVE ARTIFACT /syft syft

image-sbom:
    FROM +docker
    WORKDIR /build
    ARG TAG
    ARG FLAVOR
    ARG VARIANT
    COPY +syft/syft /usr/bin/syft
    RUN syft / -o json=sbom.syft.json -o spdx-json=sbom.spdx.json
    SAVE ARTIFACT /build/sbom.syft.json sbom.syft.json AS LOCAL build/${VARIANT}-${FLAVOR}-${TAG}-sbom.syft.json
    SAVE ARTIFACT /build/sbom.spdx.json sbom.spdx.json AS LOCAL build/${VARIANT}-${FLAVOR}-${TAG}-sbom.spdx.json

ipxe-iso:
    FROM ubuntu
    ARG ipxe_script
    RUN apt update
    RUN apt install -y -o Acquire::Retries=50 \
                           mtools syslinux isolinux gcc-arm-none-eabi git make gcc liblzma-dev mkisofs xorriso
                           # jq docker
    WORKDIR /build
    ARG ISO_NAME=${OS_ID}
    RUN git clone https://github.com/ipxe/ipxe
    IF [ "$ipxe_script" = "" ]
        COPY +netboot/ipxe /build/ipxe/script.ipxe
    ELSE
        COPY $ipxe_script /build/ipxe/script.ipxe
    END
    RUN cd ipxe/src && make EMBED=/build/ipxe/script.ipxe
    SAVE ARTIFACT /build/ipxe/src/bin/ipxe.iso iso AS LOCAL build/${ISO_NAME}-ipxe.iso.ipxe
    SAVE ARTIFACT /build/ipxe/src/bin/ipxe.usb usb AS LOCAL build/${ISO_NAME}-ipxe-usb.img.ipxe


## Security targets
trivy:
    FROM aquasec/trivy
    SAVE ARTIFACT /usr/local/bin/trivy /trivy

trivy-scan:
    ARG SEVERITY=CRITICAL
    FROM +docker
    COPY +trivy/trivy /trivy
    RUN /trivy filesystem --severity $SEVERITY --exit-code 1 --no-progress /

linux-bench:
    ARG GO_VERSION
    FROM golang:$GO_VERSION
    GIT CLONE https://github.com/aquasecurity/linux-bench /linux-bench-src
    RUN cd /linux-bench-src && CGO_ENABLED=0 go build -o linux-bench . && mv linux-bench /
    SAVE ARTIFACT /linux-bench /linux-bench

# The target below should run on a live host instead.
# However, some checks are relevant as well at container level.
# It is good enough for a quick assessment.
linux-bench-scan:
    FROM +docker
    GIT CLONE https://github.com/aquasecurity/linux-bench /build/linux-bench
    WORKDIR /build/linux-bench
    COPY +linux-bench/linux-bench /build/linux-bench/linux-bench
    RUN /build/linux-bench/linux-bench

# Generic targets
# usage e.g. ./earthly.sh +datasource-iso --CLOUD_CONFIG=tests/assets/qrcode.yaml
datasource-iso:
  ARG OSBUILDER_IMAGE
  ARG CLOUD_CONFIG
  FROM $OSBUILDER_IMAGE
  RUN zypper in -y mkisofs
  WORKDIR /build
  RUN touch meta-data
  COPY ./${CLOUD_CONFIG} user-data
  RUN cat user-data
  RUN mkisofs -output ci.iso -volid cidata -joliet -rock user-data meta-data
  SAVE ARTIFACT /build/ci.iso iso.iso AS LOCAL build/datasource.iso

# usage e.g. ./earthly.sh +run-qemu-tests --FLAVOR=alpine --FROM_ARTIFACTS=true
run-qemu-tests:
    FROM opensuse/leap
    WORKDIR /test
    RUN zypper in -y qemu-x86 qemu-arm qemu-tools go
    ARG FLAVOR
    ARG TEST_SUITE=autoinstall-test
    ARG FROM_ARTIFACTS
    ENV FLAVOR=$FLAVOR
    ENV SSH_PORT=60022
    ENV CREATE_VM=true
    ARG CLOUD_CONFIG="/tests/tests/assets/autoinstall.yaml"
    ENV USE_QEMU=true

    ENV GOPATH="/go"

    ENV CLOUD_CONFIG=$CLOUD_CONFIG

    IF [ "$FROM_ARTIFACTS" = "true" ]
        COPY . .
        ENV ISO=/test/build/kairos.iso
        ENV DATASOURCE=/test/build/datasource.iso
    ELSE
        COPY ./tests .
        COPY +iso/kairos.iso kairos.iso
        COPY ( +datasource-iso/iso.iso --CLOUD_CONFIG=$CLOUD_CONFIG) datasource.iso
        ENV ISO=/test/kairos.iso
        ENV DATASOURCE=/test/datasource.iso
    END


    RUN go install -mod=mod github.com/onsi/ginkgo/v2/ginkgo
    ENV CLOUD_INIT=$CLOUD_CONFIG

    RUN PATH=$PATH:$GOPATH/bin ginkgo --label-filter "$TEST_SUITE" --fail-fast -r ./tests/

edgevpn:
    ARG EDGEVPN_VERSION=latest
    FROM quay.io/mudler/edgevpn:$EDGEVPN_VERSION
    SAVE ARTIFACT /usr/bin/edgevpn /edgevpn

# usage e.g.
# ./earthly.sh +run-proxmox-tests --PROXMOX_USER=root@pam --PROXMOX_PASS=xxx --PROXMOX_ENDPOINT=https://192.168.1.72:8006/api2/json --PROXMOX_ISO=/test/build/kairos-opensuse-v0.0.0-79fd363-k3s.iso --PROXMOX_NODE=proxmox
run-proxmox-tests:
    FROM golang:alpine
    WORKDIR /test
    RUN apk add xorriso
    ARG FLAVOR
    ARG TEST_SUITE=proxmox-ha-test
    ARG FROM_ARTIFACTS
    ARG PROXMOX_USER
    ARG PROXMOX_PASS
    ARG PROXMOX_ENDPOINT
    ARG PROXMOX_STORAGE=local
    ARG PROXMOX_ISO
    ARG PROXMOX_NODE
    ENV GOPATH="/go"

    RUN go install -mod=mod github.com/onsi/ginkgo/v2/ginkgo

    COPY +edgevpn/edgevpn /usr/bin/edgevpn
    COPY . .
    RUN PATH=$PATH:$GOPATH/bin ginkgo --label-filter "$TEST_SUITE" --fail-fast -r ./tests/e2e/

lint:
    BUILD +hadolint
    BUILD +renovate-validator
    BUILD +shellcheck-lint
    BUILD +golangci-lint
    BUILD +yamllint

hadolint:
  FROM hadolint/hadolint:${HADOLINT_VERSION}
  COPY . /work
  WORKDIR /work
  RUN find . -name "Dockerfile*" -print  | xargs -r -n1 hadolint

renovate-validator:
  FROM renovate/renovate
  COPY . /work
  WORKDIR /work
  ENV RENOVATE_VERSION="35"
  RUN renovate-config-validator

shellcheck-lint:
  FROM koalaman/shellcheck-alpine:${SHELLCHECK_VERSION}
  COPY . /work
  WORKDIR /work
  RUN find . -name "*.sh" -print  | xargs -r -n1 shellcheck

golangci-lint:
  FROM golangci/golangci-lint:${GOLANGCILINT_VERSION}
  COPY . /work
  WORKDIR /work
  RUN golangci-lint run --timeout 360s

yamllint:
  FROM cytopia/yamllint
  COPY . /work
  WORKDIR /work
  RUN find . -name "*.yml" -or -name "*.yaml" -print  | xargs -r -n1
