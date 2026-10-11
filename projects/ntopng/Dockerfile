# OSS Scanner build for ntopng (https://github.com/ntop/ntopng).
#
# This file lives next to project.yaml in the oss-scanner repository. The scanner copies it into a fresh clone of the
# ntopng dev branch and builds it with that clone as the context.
#
# ntopng links against nDPI, which it expects as a sibling checkout (../nDPI), so nDPI is cloned and built at /nDPI
# first. ntopng is then built in /src with debug info and frame pointers; the binary is /src/ntopng.
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates build-essential git bison flex libxml2-dev libpcap-dev libtool libtool-bin rrdtool librrd-dev \
      autoconf pkg-config automake autogen redis-server wget libsqlite3-dev libhiredis-dev libmaxminddb-dev \
      libcurl4-openssl-dev libpango1.0-dev libcairo2-dev libnetfilter-queue-dev zlib1g-dev libssl-dev libcap-dev \
      libnetfilter-conntrack-dev libreadline-dev libjson-c-dev libldap2-dev rename libsnmp-dev libexpat1-dev \
      libradcli-dev libzmq3-dev curl jq libnl-genl-3-dev libgcrypt20-dev cmake libzstd-dev libmysqlclient-dev \
      libpng-dev python3 gdb valgrind lua5.4 \
    && rm -rf /var/lib/apt/lists/*

# nDPI must sit next to the ntopng checkout
RUN git clone --depth 1 https://github.com/ntop/nDPI.git /nDPI \
    && cd /nDPI && ./autogen.sh && ./configure CFLAGS="-g -fno-omit-frame-pointer" \
    && make -j"$(nproc)"

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Third-party submodule used by the ClickHouse support; fetch it if the checkout did not include it
RUN if [ ! -f third-party/clickhouse-cpp/CMakeLists.txt ]; then \
      rm -rf third-party/clickhouse-cpp && \
      git clone --depth 1 https://github.com/ClickHouse/clickhouse-cpp.git third-party/clickhouse-cpp; \
    fi

RUN ./autogen.sh \
    && ./configure CFLAGS="-g -fno-omit-frame-pointer" CXXFLAGS="-g -fno-omit-frame-pointer" \
    && make -j"$(nproc)"

# ntopng needs its geolocation DBs only at runtime for a few features; a quick sanity check that the binary links
RUN ./ntopng --version || echo "WARNING: ntopng --version failed"
