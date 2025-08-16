# =========================
# Builder stage
# =========================
FROM alpine:3.16 AS builder
LABEL maintainer="tindy.it@gmail.com"

ARG THREADS="4"
ARG SHA=""

WORKDIR /

# 基础与开发依赖
RUN set -xe && \
    apk add --no-cache --virtual .build-tools \
      git g++ build-base linux-headers cmake python3 pkgconf wget xz && \
    apk add --no-cache --virtual .build-deps \
      curl-dev rapidjson-dev pcre2-dev yaml-cpp-dev

# ---- 构建并安装 QuickJS（启用 bignum） ----
WORKDIR /tmp
RUN wget -O quickjs.tar.xz https://bellard.org/quickjs/quickjs-2024-01-13.tar.xz && \
    tar xf quickjs.tar.xz && \
    cd quickjs-2024-01-13 && \
    make -j"${THREADS}" CONFIG_BIGNUM=y && \
    make install PREFIX=/usr CONFIG_BIGNUM=y

# ---- 安装 quickjspp.hpp（header-only） ----
WORKDIR /
RUN git clone https://github.com/ftk/quickjspp --depth=1 && \
    install -m644 quickjspp/quickjspp.hpp /usr/include

# ---- 构建并安装 libcron（静态库） ----
RUN git clone https://github.com/PerMalmberg/libcron --depth=1 && \
    cd libcron && \
    git submodule update --init && \
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make libcron -j"${THREADS}" && \
    install -m644 libcron/out/Release/liblibcron.a /usr/lib/ && \
    install -d /usr/include/libcron/ && \
    install -m644 libcron/include/libcron/* /usr/include/libcron/ && \
    install -d /usr/include/date/ && \
    install -m644 libcron/externals/date/include/date/* /usr/include/date/

# ---- 安装 toml11（header-only） ----
RUN git clone https://github.com/ToruNiina/toml11 --branch="v4.3.0" --depth=1 && \
    cd toml11 && \
    cmake -DCMAKE_CXX_STANDARD=11 . && \
    make install -j"${THREADS}"

# ---- 构建 subconverter ----
COPY . /subconverter
WORKDIR /subconverter

RUN set -xe && \
    [ -n "${SHA}" ] && sed -i 's/\(v[0-9]\+\.[0-9]\+\.[0-9]\+\)/\1-'"${SHA}"'/' src/version.h; \
    python3 -m ensurepip && \
    python3 -m pip install --no-cache-dir gitpython && \
    python3 scripts/update_rules.py -c scripts/rules_config.conf && \
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make -j"${THREADS}"

# =========================
# Runtime stage
# =========================
FROM alpine:3.16
LABEL maintainer="tindy.it@gmail.com"

# 运行时依赖（含 libstdc++）
RUN apk add --no-cache --virtual subconverter-deps pcre2 libcurl yaml-cpp libstdc++

# 放入二进制与基础规则
COPY --from=builder /subconverter/subconverter /usr/bin/
COPY --from=builder /subconverter/base /base/

# 时区（可按需修改）
ENV TZ=Africa/Abidjan
RUN ln -sf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

WORKDIR /base
EXPOSE 25500/tcp

# 用 JSON 形式的 CMD（避免信号处理警告）
CMD ["subconverter"]
