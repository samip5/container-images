#
# NOTE: qbittorrent >= 4.3.0 requires libtorrent-rasterbar >= 1.2
# hence all the scattered logic written below
#

# hadolint ignore=DL3007
FROM ghcr.io/k8s-at-home/ubuntu:latest as builder

ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG VERSION

# hadolint ignore=DL3002
USER root

WORKDIR /tmp

# Install system dependencies
# hadolint ignore=DL3008
RUN \
  apt-get -qq update \
  && \
  apt-get -qq install -y --no-install-recommends \
    build-essential \
    cmake \
    git \
    libboost-chrono-dev \
    libboost-dev \
    libboost-random-dev \
    libboost-system-dev \
    libgeoip-dev \
    libqt5svg5-dev \
    libssl-dev \
    ninja-build \
    pkg-config \
    qtbase5-dev \
    qttools5-dev \
    zlib1g-dev

# Install lintorrent-rasterbar dependencies
RUN \
  case "${TARGETPLATFORM}" in \
    'linux/amd64') \
      export LIBTORRENT_ARCH='amd64'; \
      ;; \
    'linux/arm64') \
      export LIBTORRENT_ARCH='arm64'; \
      ;; \
  esac \
  && \
  export LIBTORRENT_DEB="libtorrent-rasterbar10_1.2.14+git20210727.ba56dd1e23-1ppa1~20.04_${LIBTORRENT_ARCH}.deb" \
  && export LIBTORRENT_DEV_DEB="libtorrent-rasterbar-dev_1.2.14+git20210727.ba56dd1e23-1ppa1~20.04_${LIBTORRENT_ARCH}.deb" \
  && \
  curl -fsSL -o libtorrent-rasterbar.deb \
    "https://launchpad.net/~qbittorrent-team/+archive/ubuntu/qbittorrent-stable/+files/${LIBTORRENT_DEB}" \
  && dpkg -i libtorrent-rasterbar.deb \
  && rm -rf libtorrent-rasterbar.deb \
  && \
  curl -fsSL -o libtorrent-rasterbar-dev.deb \
    "https://launchpad.net/~qbittorrent-team/+archive/ubuntu/qbittorrent-stable/+files/${LIBTORRENT_DEV_DEB}" \
  && dpkg -i libtorrent-rasterbar-dev.deb \
  && rm -rf libtorrent-rasterbar-dev.deb

# Compile qbitorrent
RUN \
  git clone --depth 1 -b "release-${VERSION}" https://github.com/qbittorrent/qBittorrent.git . \
  && ./configure "${EXTRA_CFG_ARG}" --disable-gui --disable-stacktrace --disable-qt-dbus \
  && make \
  && make install \
  && strip /usr/local/bin/qbittorrent-nox -o /usr/local/bin/qbittorrent-nox-stripped

# hadolint ignore=DL3007
FROM ghcr.io/k8s-at-home/ubuntu:latest

ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG VERSION

# Proper way to set config directory
ENV HOME=/config \
    XDG_CONFIG_HOME=/config \
    XDG_DATA_HOME=/config \
    WEBUI_PORT=8080

USER root

COPY --from=builder /usr/local/bin/qbittorrent-nox-stripped /app/qbittorrent-nox

# hadolint ignore=DL3008,DL3015,SC2086
RUN \
  case "${TARGETPLATFORM}" in \
    'linux/amd64') \
      export LIBTORRENT_ARCH='amd64'; \
      ;; \
    'linux/arm64') \
      export LIBTORRENT_ARCH='arm64'; \
      ;; \
  esac \
  && \
  export LIBTORRENT_DEB="libtorrent-rasterbar10_1.2.14+git20210727.ba56dd1e23-1ppa1~20.04_${LIBTORRENT_ARCH}.deb" \
  && \
  curl -fsSL -o /tmp/libtorrent-rasterbar.deb \
    "https://launchpad.net/~qbittorrent-team/+archive/ubuntu/qbittorrent-stable/+files/${LIBTORRENT_DEB}" \
  && dpkg -i /tmp/libtorrent-rasterbar.deb \
  && \
  apt-get -qq update \
  && \
  apt-get install -y \
    geoip-bin \
    libqt5network5 \
    libqt5xml5 \
    p7zip-full \
    python3 \
    unrar \
    unzip \
  && \
  apt-get autoremove -y \
  && apt-get clean \
  && \
  rm -rf \
    /tmp/* \
    /var/lib/apt/lists/* \
    /var/tmp/ \
  && chown -R kah:kah /app \
  && chmod -R u=rwX,go=rX /app \
  && printf "umask %d" "${UMASK}" >> /etc/bash.bashrc \
  && update-ca-certificates

USER kah

EXPOSE 6881 6881/udp ${WEBUI_PORT}

COPY ./apps/qbittorrent/qBittorrent.conf /app/qBittorrent.conf

COPY ./apps/qbittorrent/shim/config.py /shim/config.py

COPY ./apps/qbittorrent/entrypoint.sh /entrypoint.sh

CMD ["/entrypoint.sh"]
