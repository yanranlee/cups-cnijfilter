FROM docker.1ms.run/olbat/cupsd:latest

# 配置国内源，仅安装解包必需的 tar（CUPS 及底层基础运行库镜像内已具备）
RUN rm -f /etc/apt/sources.list.d/* && \
    echo "deb http://mirrors.aliyun.com/debian/ testing main contrib non-free non-free-firmware" > /etc/apt/sources.list && \
    apt-get update && \
    apt-get install -y --no-install-recommends tar && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /tmp

# 复制本地驱动包
COPY cnijfilter-mp280series-3.40-1-deb.tar.gz /tmp/

# 解包并安装，建立旧版库兼容软链接
RUN tar -zxvf cnijfilter-mp280series-3.40-1-deb.tar.gz && \
    cd cnijfilter-mp280series-3.40-1-deb/packages && \
    dpkg -i --force-depends cnijfilter-common_*_amd64.deb cnijfilter-mp280series_*_amd64.deb || true && \
    chmod 755 /usr/lib/cups/filter/pstocanonij 2>/dev/null || true && \
    chmod 755 /usr/lib/cups/filter/cifmp280 2>/dev/null || true && \
    # 动态自动建立 libtiff.so.4 和 libpng12.so.0 的兼容软链接
    TIFF_LIB=$(find /usr/lib /lib -name "libtiff.so*" 2>/dev/null | head -n 1) && \
    PNG_LIB=$(find /usr/lib /lib -name "libpng*.so*" 2>/dev/null | head -n 1) && \
    [ -n "$TIFF_LIB" ] && ln -sf "$TIFF_LIB" /usr/lib/libtiff.so.4 || true && \
    [ -n "$PNG_LIB" ] && ln -sf "$PNG_LIB" /usr/lib/libpng12.so.0 || true && \
    ldconfig && \
    cd / && rm -rf /tmp/*

WORKDIR /