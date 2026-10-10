FROM docker.1ms.run/olbat/cupsd:latest

WORKDIR /tmp

# copy cnijfilter
COPY cnijfilter-mp280series-3.40-1-deb.tar.gz /tmp/

# Extract and install, create legacy library compatibility symlinks
RUN tar -zxvf cnijfilter-mp280series-3.40-1-deb.tar.gz && \
    cd cnijfilter-mp280series-3.40-1-deb/packages && \
    dpkg -i --force-depends cnijfilter-common_*_amd64.deb cnijfilter-mp280series_*_amd64.deb || true && \
    chmod 755 /usr/lib/cups/filter/pstocanonij 2>/dev/null || true && \
    chmod 755 /usr/lib/cups/filter/cifmp280 2>/dev/null || true && \
    # Dynamically create compatibility symlinks for libtiff.so.4 and libpng12.so.0
    TIFF_LIB=$(find /usr/lib /lib -name "libtiff.so*" 2>/dev/null | head -n 1) && \
    PNG_LIB=$(find /usr/lib /lib -name "libpng*.so*" 2>/dev/null | head -n 1) && \
    [ -n "$TIFF_LIB" ] && ln -sf "$TIFF_LIB" /usr/lib/libtiff.so.4 || true && \
    [ -n "$PNG_LIB" ] && ln -sf "$PNG_LIB" /usr/lib/libpng12.so.0 || true && \
    ldconfig && \
    cd / && rm -rf /tmp/*

WORKDIR /