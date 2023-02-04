VERSION 0.6
# Note the base image needs to have dracut.
# TODO: This needs to come from pre-built kernels in c3os repos, immucore included.
# Framework images should use our initrd
ARG FLAVOR=core-opensuse-leap
ARG BASE_IMAGE=quay.io/kairos/$FLAVOR
ARG OSBUILDER_IMAGE=quay.io/kairos/osbuilder-tools
ARG ISO_NAME=$FLAVOR-immucore

ARG GO_VERSION=1.18
ARG GOLINT_VERSION=v1.47.3

version:
    FROM alpine
    RUN apk add git
    COPY . ./
    RUN --no-cache echo $(git describe --always --tags --dirty) > VERSION
    ARG VERSION=$(cat VERSION)
    SAVE ARTIFACT VERSION VERSION

golang-image:
    ARG GO_VERSION
    FROM golang:$GO_VERSION
    WORKDIR /build
    COPY go.mod go.sum ./
    RUN go mod download

go-deps:
    ARG GO_VERSION
    FROM golang:$GO_VERSION
    WORKDIR /build
    COPY go.mod go.sum ./
    RUN go mod download
    RUN apt-get update
    SAVE ARTIFACT go.mod AS LOCAL go.mod
    SAVE ARTIFACT go.sum AS LOCAL go.sum

test:
    FROM +golang-image
    WORKDIR /build
    RUN go install -mod=mod github.com/onsi/ginkgo/v2/ginkgo
    COPY . .
    RUN ginkgo run --race --fail-fast --slow-spec-threshold 30s --covermode=atomic --coverprofile=coverage.out -p -r ./...
    SAVE ARTIFACT coverage.out AS LOCAL coverage.out

lint:
    ARG GOLINT_VERSION
    FROM golangci/golangci-lint:$GOLINT_VERSION
    WORKDIR /build
    COPY . .
    RUN golangci-lint run

build-immucore:
    FROM +golang-image
    COPY +version/VERSION ./
    ARG VERSION=$(cat VERSION)
    WORKDIR /work
    COPY . /work
    RUN CGO_ENABLED=0 go build -o immucore -ldflags "-X main.Version=$VERSION"
    SAVE ARTIFACT /work/immucore AS LOCAL build/immucore-$VERSION

build-dracut:
    FROM $BASE_IMAGE
    COPY +version/VERSION ./
    ARG VERSION=$(cat VERSION)
    COPY . /work
    COPY +build-immucore/immucore /usr/bin/immucore
    WORKDIR /work
    RUN cp -r dracut/28immucore /usr/lib/dracut/modules.d
    RUN cp dracut/dracut.conf /etc/dracut.conf.d/10-immucore.conf
    RUN kernel=$(ls /lib/modules | head -n1) && \
        dracut -f "/boot/initrd-${kernel}" "${kernel}" && \
        ln -sf "initrd-${kernel}" /boot/initrd
    ARG INITRD=$(readlink -f /boot/initrd)
    SAVE ARTIFACT $INITRD Initrd AS LOCAL build/initrd-$VERSION

image:
    FROM $BASE_IMAGE
    COPY +version/VERSION ./
    ARG VERSION=$(cat VERSION)
    ARG INITRD=$(readlink -f /boot/initrd)
    COPY +build-dracut/Initrd $INITRD
    # For initrd use
    COPY +build-immucore/immucore /usr/bin/immucore
    RUN ln -s /usr/lib/systemd/systemd /init
    SAVE IMAGE $FLAVOR-immucore:$VERSION

image-rootfs:
    FROM +image
    SAVE ARTIFACT --keep-own /. rootfs

grub-files:
    FROM alpine
    RUN apk add wget
    RUN wget https://raw.githubusercontent.com/c3os-io/c3os/master/overlay/files-iso/boot/grub2/grub.cfg -O grub.cfg
    SAVE ARTIFACT --keep-own grub.cfg grub.cfg

iso:
    FROM $OSBUILDER_IMAGE
    ARG ISO_NAME
    COPY +version/VERSION ./
    ARG VERSION=$(cat VERSION)
    WORKDIR /build
    COPY --keep-own +grub-files/grub.cfg /build/files-iso/boot/grub2/grub.cfg
    COPY --keep-own +image-rootfs/rootfs /build/rootfs
    RUN /entrypoint.sh --name $ISO_NAME --debug build-iso --squash-no-compression --date=false --local --overlay-iso /build/files-iso --output /build/ dir:/build/rootfs
    SAVE ARTIFACT /build/$ISO_NAME.iso iso AS LOCAL build/$ISO_NAME-$VERSION.iso
    SAVE ARTIFACT /build/$ISO_NAME.iso.sha256 sha256 AS LOCAL build/$ISO_NAME-$VERSION.iso.sha256