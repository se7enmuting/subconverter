# ===================== builder =====================
FROM alpine:3.16 AS builder
LABEL maintainer="tindy.it@gmail.com"

ARG THREADS="4"
ARG SHA=""

# 基础工具 + 依赖
RUN set -xe && \
  apk add --no-cache --virtual .build-tools \
    git g++ build-base linux-headers cmake python3 pkgconf wget && \
  apk add --no-cache --virtual .build-deps \
    curl-dev rapidjson-dev pcre2-dev yaml-cpp-dev

WORKDIR /tmp

# ---- 构建 QuickJS（启用 bignum，并固定到含 JS_GetClassID 的提交）----
# 说明：我们用 c50de13b1573...（2024-02-21）这个提交：有 JS_GetClassID，且仍有 BigDecimal/BigFloat
ENV QUICKJS_COMMIT=c50de13b1573735c57e918f2739428b82dd2481f
RUN set -xe && \
  git clone https://github.com/bellard/quickjs && \
  cd quickjs && \
  # 保险做法：只拉需要的那一个提交（防止 shallow 导致 checkout 失败）
  git fetch --depth=1 origin ${QUICKJS_COMMIT} && \
  git checkout FETCH_HEAD && \
  make -j"${THREADS}" CONFIG_BIGNUM=y && \
  make install PREFIX=/usr CONFIG_BIGNUM=y

# ---- libcron ----
RUN set -xe && \
  git clone --depth=1 https://github.com/PerMalmberg/libcron && \
  cd libcron && \
  git submodule update --init && \
  cmake -DCMAKE_BUILD_TYPE=Release . && \
  make libcron -j "${THREADS}" && \
  install -m644 libcron/out/Release/liblibcron.a /usr/lib/ && \
  install -d /usr/include/libcron/ && \
  install -m644 libcron/include/libcron/* /usr/include/libcron/ && \
  install -d /usr/include/date/ && \
  install -m644 libcron/externals/date/include/date/* /usr/include/date/

# ---- toml11 ----
RUN set -xe && \
  git clone --depth=1 --branch v4.3.0 https://github.com/ToruNiina/toml11 && \
  cd toml11 && \
  cmake -DCMAKE_CXX_STANDARD=11 . && \
  make install -j "${THREADS}"

# ---- 拷贝源码并编译 subconverter ----
WORKDIR /subconverter
COPY . /subconverter

RUN set -xe && \
  [ -n "${SHA}" ] && sed -i 's/\(v[0-9]\+\.[0-9]\+\.[0-9]\+\)/\1-'"${SHA}"'/' src/version.h; \
  python3 -m ensurepip && \
  python3 -m pip install --no-cache-dir gitpython && \
  python3 scripts/update_rules.py -c scripts/rules_config.conf && \
  # 关键：用宏把 Unhandled 版本的 API 映射到旧名，兼容头文件实现
  cmake -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_CXX_FLAGS="-DJS_SetHostUnhandledPromiseRejectionTracker=JS_SetHostPromiseRejectionTracker" \
        . && \
  make -j"${THREADS}"

# ===================== runtime =====================
FROM alpine:3.16
LABEL maintainer="tindy.it@gmail.com"

RUN apk add --no-cache --virtual subconverter-deps pcre2 libcurl yaml-cpp

COPY --from=builder /subconverter/subconverter /usr/bin/
COPY --from=builder /subconverter/base /base/

ENV TZ=Africa/Abidjan
RUN ln -sf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

WORKDIR /base
CMD ["subconverter"]

EXPOSE 25500/tcp
