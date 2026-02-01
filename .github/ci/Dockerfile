ARG DOCKER_ARCH
ARG DEBIAN_VERSION
ARG DOCKER_REPO
FROM --platform=linux/${DOCKER_ARCH} ${DOCKER_REPO}debian:${DEBIAN_VERSION} AS build_env

RUN apt-get -y update && apt-get -y install gnupg2 wget ca-certificates

# Add RPI packages (the whole base system, since RPI ships its own GCC)
ARG DEBIAN_VERSION
ARG BUILD_TYPE="generic"
RUN [ "$BUILD_TYPE" != "raspi" ] || \
  ( \
    install_keyring() { \
      local repo_url=$1; \
      local pkg_name=$2; \
      echo "Fetching latest $pkg_name from $repo_url..."; \
      # Scrape the directory listing for the latest .deb version
      local deb_file=$(wget -qO - "$repo_url" | \
        grep -o "${pkg_name}_[0-9a-zA-Z\.\+\-]*_all.deb" | \
        sort -V | tail -n 1); \
      if [ -z "$deb_file" ]; then echo "Failed to find deb file for $pkg_name"; exit 1; fi; \
      wget -qO "/tmp/${pkg_name}.deb" "$repo_url/$deb_file" && \
      dpkg -i "/tmp/${pkg_name}.deb" && \
      rm "/tmp/${pkg_name}.deb"; \
    }; \
    \
    install_keyring "http://archive.raspberrypi.org/debian/pool/main/r/raspberrypi-archive-keyring" "raspberrypi-archive-keyring" && \
    \
    if [ "$(dpkg --print-architecture)" = "armhf" ]; then \
      install_keyring "http://archive.raspbian.org/raspbian/pool/main/r/raspbian-archive-keyring" "raspbian-archive-keyring" && \
      echo "deb http://raspbian.raspberrypi.org/raspbian/ $DEBIAN_VERSION main contrib non-free rpi" > /etc/apt/sources.list; \
    fi && \
    \
    # 4. Add the Repo Source and Update
    echo "deb http://archive.raspberrypi.org/debian/ $DEBIAN_VERSION main" > /etc/apt/sources.list.d/raspi.list && \
    apt-get -y update && \
    apt-get -y install libcamera-dev liblivemedia-dev \
  )

# Default packages
RUN apt-get -y install build-essential xxd cmake ccache git-core pkg-config \
  libavformat-dev libavutil-dev libavcodec-dev libssl-dev v4l-utils debhelper

FROM build_env AS build
ADD / /src
WORKDIR /src
RUN git clean -ffdx
RUN git submodule update --init --recursive --recommend-shallow
RUN git submodule foreach --recursive git clean -ffdx

FROM build AS deb_make
ARG GIT_VERSION
ARG BUILD_TYPE="generic"
ENV DEB_BUILD_PROFILES="$BUILD_TYPE"

RUN apt-get build-dep -y $PWD
RUN . /etc/os-release && \
  export RELEASE_SUFFIX="$VERSION_CODENAME" && \
  dpkg-buildpackage -us -uc -b

RUN mkdir -p /deb && mv ../*.deb /deb/
