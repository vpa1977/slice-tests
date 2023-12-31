ARG UBUNTU_RELEASE=22.04
ARG USER=app
ARG UID=101
ARG GROUP=app
ARG GID=101

FROM golang:1.18 AS chisel
ARG UBUNTU_RELEASE
RUN git clone -b busybox-slice https://github.com/vpa1977/chisel-releases /opt/chisel-releases \
    && git clone --depth 1 -b main https://github.com/canonical/chisel /opt/chisel
WORKDIR /opt/chisel
RUN go generate internal/deb/version.go \
    && go build ./cmd/chisel

FROM public.ecr.aws/ubuntu/ubuntu:$UBUNTU_RELEASE@sha256:2c1168b31e636c7ab22598bfaefa6de63d1806a908d0c39bd402277881356fab AS builder
SHELL ["/bin/bash", "-oeux", "pipefail", "-c"]
RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y \
        binutils \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*
COPY --from=chisel /opt/chisel/chisel /usr/bin/

### BOILERPLATE END ###
FROM builder as sliced-deps
ARG USER
ARG UID
ARG GROUP
ARG GID
SHELL ["/bin/bash", "-oeux", "pipefail", "-c"]
COPY --from=chisel /opt/chisel-releases /opt/chisel-releases
RUN mkdir -p /rootfs \
    && chisel cut --release /opt/chisel-releases --root /rootfs \
    busybox_bins
RUN install -d -m 0755 -o $UID -g $GID /rootfs/home/$USER \
    && mkdir -p /rootfs/etc \
    && echo -e "root:x:0:\n$GROUP:x:$GID:" >/rootfs/etc/group \
    && echo -e "root:x:0:0:root:/root:/noshell\n$USER:x:$UID:$GID::/home/$USER:/noshell" >/rootfs/etc/passwd
FROM scratch
ARG USER
ARG UID
ARG GID
USER $UID:$GID
COPY --from=sliced-deps /rootfs /
# Workaround for https://github.com/moby/moby/issues/38710
COPY --from=sliced-deps --chown=$UID:$GID /rootfs/home/$USER /home/$USER
ENTRYPOINT ["/bin/busybox"]
CMD ["sh"]
